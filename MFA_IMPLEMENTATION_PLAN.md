# Multi-Factor Authentication Implementation Plan for Apache OFBiz

**Status:** Phase 0, Phase 1, and the scoped Phase 2 implementation and focused acceptance checks are complete for common, Party/securityext, standard browser login, and ecommerce storefront-wrapper flows. `UserLoginTests` passes 9/9 on MySQL; the ecommerce MFA response-propagation unit test passes 1/1. Browser-driven enrollment/login and the inventoried alternate identity-entry points have not been verified or brought under MFA. MFA remains disabled by default, and no required-authenticator group should be enabled until the deferred routes below have explicit policy handling and tests. Verification used Gradle 8.14.5 offline with unrelated Node setup tasks excluded. Logs confirmed MySQL Connector/J 9.6.0 and MySQL 8.4.11. Startup additively created `USER_AUTHENTICATION_EVENT`, `USER_LOGIN_AUTH_CHALLENGE`, and `USER_LOGIN_AUTH_FACTOR`, and added `SECURITY_GROUP.REQUIRE_AUTH_FACTOR`; no clean, drop, or reset ran. The Apache OFBiz default, test, and no-ECA delegators map to the configured local MySQL datasources.

**Branch:** `feature/mfa-implementation`

**Base:** Apache OFBiz framework trunk at `3cf93ad862` and plugins trunk at `1f036f65e` (25 September 2026)

**Scope:** Framework, applications, and plugins in this checkout

## 1. Executive recommendation

The two MFA proposals can be implemented as one Apache OFBiz® feature. Use Proposal 1 for the factor types, enrollment concepts, policy goal, and login challenge flow. Use Proposal 2 to correct the authentication boundary and to define the protections around password changes, alternate login methods, challenges, auditing, and REST token issuance.

The first engineering task should separate primary credential verification from final successful-login processing. Apache OFBiz currently records successful-login state inside `LoginServices.userLogin()` before the web session is established by `LoginWorker.doMainLogin()`. A challenge inserted only near the end of `LoginWorker.login()` would stop the normal session from being created, but it would not undo successful-login state already recorded by the service.

No MFA implementation was found in the synced Apache OFBiz framework or plugins when searched for `UserLoginAuthFactor`, `requireAuthFactor`, and MFA challenge identifiers. This plan describes proposed work, not an existing capability.

## 2. Documents reviewed and how they fit together

The source documents are:

- [Proposal 1 PDF](../1_OFBiz-MFA-Proposal.pdf) and [Proposal 1 DOCX](../1_OFBiz-MFA-Proposal.docx): the base RFC. It proposes TOTP, backup codes, email/SMS factors, group enforcement, enrollment, web login challenge handling, rate limits, multi-tenant behavior, and rollout.
- [Proposal 2 PDF](../2_OFBiz_MFA_Proposal.pdf) and [Proposal 2 DOCX](../2_OFBiz_MFA_Proposal.docx): the security addendum. It identifies the early login-state commit, password mutation before MFA, overly broad SSO/JWT assumptions, weak challenge binding, incomplete auditing, tenant/session lifecycle, and REST integration issues.

The two documents are complementary. Proposal 2 does not replace the feature scope in Proposal 1; it changes important ordering and security requirements for implementing that scope in Apache OFBiz.

## 3. Moqui reference implementation findings

The linked `Krishnauprit18/moqui-framework` repository is a public fork of `moqui/moqui-framework`. Its `master` snapshot inspected for this plan was commit `6b1974438a87fe5b1e9279de9795d0f19658f5a8` from 3 September 2026.

Relevant Moqui implementation locations:

- `framework/entity/SecurityEntities.xml`: defines `UserAuthcFactor` and `UserGroup.requireAuthcFactor`. Factor data includes a type, validity dates, an encrypted option, and a validation flag.
- `framework/service/org/moqui/impl/UserServices.xml`: implements factor requirement evaluation, factor information, TOTP verification/setup, single-use codes, email/SMS sends, and password-update checks.
- `framework/src/main/groovy/org/moqui/impl/util/MoquiShiroRealm.groovy`: evaluates the factor requirement after primary authentication and raises `SecondFactorRequiredException`.
- `framework/src/main/groovy/org/moqui/impl/context/UserFacadeImpl.groovy`: catches that exception, stores the pre-authenticated username and pending-factor flag in the web session, and returns without establishing the authenticated subject. Successful factor verification then completes login through an internal login token.
- `framework/src/test/groovy/SecurityAuthnTests.groovy`: includes positive and negative pre-authentication MFA tests. Other tests verify that factor data is not exposed by generic entity REST access.
- `ReleaseNotes.md`: records login and password-update MFA with TOTP, email/SMS codes, backup codes, and group enforcement.

### Patterns to adapt

- Evaluate MFA after primary authentication and before final authenticated session state.
- Keep a server-side pre-authentication identity; do not accept the target user identity from a public verification request.
- Centralize the “is another factor required?” decision.
- Treat enrollment as pending until the user proves possession of the new factor.
- Store TOTP secrets reversibly encrypted and single-use codes as one-way hashes.
- Bind email/SMS challenges to a factor owned by the pending user; consume successful one-time codes.
- Apply the MFA requirement to password-update flows as well as login.
- Prevent generic entity or service APIs from exposing factor secrets.

### Moqui behaviors not to copy without correction

- Its MFA verification service has a TODO for limiting code attempts. Apache OFBiz must define and enforce verification and code-send limits from the first externally usable release.
- Its factor-required service counts current factors without excluding `needsValidation='Y'`. Apache OFBiz should count only active, validated factors as satisfying a normal login requirement.
- The factor information service returns the email/phone option for display. Apache OFBiz should mask destinations and return only the minimum necessary information.
- Moqui holds pending identity in session state rather than a distinct persisted challenge record. Proposal 2's challenge binding, expiry, attempt count, status, and tenant context are appropriate extensions for concurrent challenges and better auditability.
- Moqui's Shiro exception and force-login token are framework-specific. Apache OFBiz should use its own service, controller, and session conventions rather than importing that mechanism.

## 4. Current Apache OFBiz code map

| Area | Current code | Implementation consequence |
|---|---|---|
| Primary authentication and login bookkeeping | `framework/common/src/main/java/org/apache/ofbiz/common/login/LoginServices.java`; `framework/common/servicedef/services.xml` | `userLogin` is the common authentication service and writes account state and login history on successful authentication. Extract or introduce a trusted internal primary-authentication path that can defer successful-login bookkeeping for interactive MFA. Preserve the existing service contract for service-auth callers. |
| Web login and completed session | `framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/LoginWorker.java` | `login()` invokes `userLogin`; `doMainLogin()` and `doBasicLogin()` establish the authenticated session. MFA must be completed before those finalization steps. Login currently rotates the session at the start; the completed authentication should rotate it again before the final session is established. |
| Login gate response handling | `LoginWorker.checkLogin()` and `framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java` | `checkLogin()` currently treats `error` as the failed-login result and otherwise continues as success. Add an explicit pending-MFA result and response path so the original secured request cannot proceed as authenticated. Handle AJAX requests separately. |
| Shared controller configuration | `framework/common/webcommon/WEB-INF/common-controller.xml` | Add challenge, verify, send-code, enrollment, and management mappings/views where they belong. Map the pending result from login, check-login, and AJAX check-login. Keep public pre-auth endpoints HTTPS-only, CSRF protected for browser POSTs, and backed by narrow server-side state. |
| Password change/reset | `LoginWorker.login()`, `LoginServices.updatePassword()`, `framework/common/webcommon/WEB-INF/common-controller.xml`, and `applications/securityext` | A required password change can call `updatePassword` before the proposed login MFA hook. Require MFA or an explicit recovery route before mutating the password for MFA-mandated users. Define how forgot-password reset and admin reset interact with MFA. |
| Alternate web authentication | `LoginWorker.loginUserWithUserLoginId()`, `LoginWorker.extensionConnectLogin()`, certificate/header/remote-user methods, `JWTManager.checkJWTLogin()`, and `framework/common/webcommon/WEB-INF/common-controller.xml` preprocessors | Several paths can establish a session without the standard username/password `login()` flow. `LoginWorker.login()` also accepts a `TOKEN` input, and the common controller has JWT and extension-connect preprocessors. Inventory each path and route it through the same policy decision, or explicitly keep it out of enforced scope until its assurance behavior is implemented. Do not infer MFA from a valid token or certificate alone. |
| Cross-webapp login keys and impersonation | `framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/ExternalLoginKeysManager.java` and `LoginWorker` impersonation methods | Carry authentication assurance with any trusted cross-webapp session transition. Invalidate or constrain pending MFA on logout, session replacement, tenant/delegator changes, and authentication restart. Audit impersonator and subject separately. |
| Security groups and user login model | `framework/security/entitydef/entitymodel.xml`; `framework/common/servicedef/services_security.xml` | Add a group-level MFA requirement through the existing group model and permission-checked admin services. A user with an expired group membership must not inherit the requirement. Factor eligibility must use date windows and validation state. |
| Entity encryption and credential utilities | `framework/security` and `framework/entity` | Use the entity engine's encrypted-field support for TOTP secrets. Hash recovery and delivery codes. Do not put codes, TOTP secrets, provisioning URIs, or QR data in logs or audit payloads. |
| REST and service token issuance | `framework/rest-api/src/main/java/org/apache/ofbiz/ws/rs/security/auth/HttpBasicAuthFilter.java`, `.../resources/AuthenticationResource.java`, `framework/rest-api/servicedef/services.xml`, and `framework/rest-api/src/main/java/org/apache/ofbiz/ws/rs/services/RestServices.java` | The Basic filter calls `userLogin`; `AuthenticationResource` creates access and refresh tokens at `/{apiGroupPath}/token`. Separately, `generateAuthTokenService` calls `RestServices.generateAuthToken()` to mint a token from a supplied `userLogin`. Audit who can invoke that service and whether its callers are trusted internal flows or a possible policy bypass; enforce appropriate MFA assurance at the token-issuance boundary. Preserve API-group context and review refresh-token behavior separately. |
| Plugin storefront login | `applications/securityext/src/main/java/org/apache/ofbiz/securityext/login/LoginEvents.java` and `plugins/ecommerce/webapp/ecommerce/WEB-INF/controller.xml` | Ecommerce wraps `LoginWorker.login()` and `checkLogin()` in `storeLogin()` and `storeCheckLogin()`. Propagate MFA-pending responses through those wrappers and mappings, while retaining store-role checks only after completed authentication. Review other plugin controllers for overrides or custom login events. |
| Dependency and configuration | Root `build.gradle`, `dependencies.gradle`, and `framework/security/config/security.properties` | Proposal 1 references `framework/security/build.gradle` and `framework/security/servicedef/services.xml`; those paths are not present in this synced trunk. Use the repository's actual dependency and service organization. Add defaults that leave existing installations unchanged until explicitly configured. Record third-party library license information in the repository's dependency/notice process. |

### Current REST path correction

Proposal 2 sketches `POST /auth/token`. The current Apache OFBiz route is `POST /auth/{apiGroupPath}/token`; `HttpBasicAuthFilter` authenticates Basic credentials by calling `userLogin`, and `AuthenticationResource` mints the tokens. The MFA protocol must fit that route and the API-group claim. Do not issue access or refresh tokens in the primary-credential step when the policy requires MFA.

## 5. Proposed Apache OFBiz architecture

Keep four responsibilities explicit:

1. **Primary authentication:** verify password or configured external primary credential. Record failed primary attempts under existing login controls. For a web login that may require MFA, do not yet record a completed successful login or establish an authenticated session.
2. **Authentication policy:** decide `ALLOW`, `REQUIRE_MFA`, `REQUIRE_ENROLLMENT`, or `DENY` from user, active security-group membership, authentication method, channel, and verified upstream assurance. Keep the first version small and default-disabled.
3. **MFA services:** enroll/revoke factors, create and deliver challenges, verify factors, enforce expiry and rate limits, and write security audit events. Public web events derive user, tenant, and active challenge from server-side pending state.
4. **Authentication finalization:** after policy is satisfied, update successful-login state/history exactly once, rotate the session, call the normal login tail, and run after-login events/cookies. REST adapters issue tokens only at this point.

State flow:

```text
Unauthenticated
  -> Primary verified
      -> Policy allows -> Authentication committed
      -> MFA required -> MFA pending -> Factor verified -> Authentication committed
      -> Enrollment required -> Enrollment pending -> Factor verified -> Authentication committed
```

If MFA is disabled or not required, preserve current successful behavior through the same finalization function. A caller-controlled request parameter must not be able to select deferred or finalization behavior.

### Data model proposal

- `UserLoginAuthFactor`: owner, factor type, label, validity dates, lifecycle/validation state, last-used/last-validated data, and type-appropriate encrypted secret or destination. Enforce type-specific handling if a shared value column is chosen. Only active validated factors satisfy login policy.
- `UserLoginAuthChallenge`: opaque challenge ID, user login ID, selected factor, tenant/delegator, session-binding hash, created/expiry times, attempt count/limit, lifecycle status, and code hash only for email/SMS challenges. Do not persist raw session IDs or plaintext codes.
- `SecurityGroup.requireAuthFactor`: group-level requirement, managed only through authorized security administration. Evaluate only active group membership.
- `UserAuthenticationEvent` (or equivalent append-only audit record): primary success, MFA requirement, challenge send, verification success/failure, recovery use, enrollment/revocation, and final login commit. Never record secrets or submitted codes. Keep existing `UserLoginHistory` semantics compatible.
- A separate attempt entity is optional if challenge counters and audit events provide safe atomic limiting. Decide this during schema design; avoid duplicate counter sources.

Use Entity Engine relations and per-tenant delegators. Ensure concurrent verification cannot consume one recovery code twice or exceed the challenge limit through a race. If replay within a valid TOTP time step is to be prevented, persist and atomically enforce the last accepted time step per TOTP factor.

### Factor and enrollment behavior

- **TOTP:** generate a cryptographically strong secret; show a transient `otpauth://` URI/QR once; require a successful code before activating the factor; encrypt the secret at rest; verify RFC 6238 codes with a documented clock window.
- **Recovery codes:** generate with `SecureRandom`; store only one-way hashes; display once; consume atomically; rotate and revoke the remaining set when regenerated.
- **Email/SMS:** use a common challenge lifecycle, short expiry, one-time hashed code, masked destination, ownership validation, separate send throttles, and generic public errors. Confirm supported email configuration and an SMS delivery adapter before committing SMS to the first release.
- **Forced enrollment:** a required group with no active validated factor must reach an enrollment-only state, not normal application access. Provide a documented administrator recovery path so an unavailable email/SMS provider or lost device does not cause silent bypass.
- **Self-service changes:** factor add/remove and recovery-code rotation require an authenticated user plus recent re-authentication/MFA. Administrative reset requires a privileged permission, reason, and audit event.

## 6. Authentication channel and rollout scope

Before enabling a group-level policy, complete a channel inventory and select a policy for each path:

| Channel | Required design decision |
|---|---|
| Browser interactive login | Username/password and the `TOKEN` input accepted by `LoginWorker.login()` are both primary-credential routes. Both must reach the MFA policy; token validity alone must not skip a required second factor. Use deferred successful-login commit and a no-session-before-MFA invariant. |
| Required password change | Authenticate primary, complete required MFA/recovery, then change the password and finalize. Prevent the password update from becoming an MFA bypass. |
| Forgot/reset password | Define whether the factor is required, how recovery is proven, and what administrative reset does. Audit every recovery path. |
| Header/remote user and X.509 | Establish whether the upstream credential meets policy. Only bypass local MFA when an administrator explicitly configures and trusts that assurance. |
| JWT and external login key | Cover `JWTManager.checkJWTLogin()`, `LoginWorker.login()`'s `TOKEN` input, `extensionConnectLogin()`, and external login keys. Token/key validity proves origin/validity, not MFA. Carry verified assurance where available; otherwise require local step-up or keep the path outside enforced policy until supported. |
| REST and service token issuance | Cover the Basic-auth token resource and `generateAuthTokenService` / `RestServices.generateAuthToken()`. Primary credentials may create only an MFA-pending challenge. Audit service callers and gate issuance on suitable assurance; a second request verifies the challenge and only then creates access/refresh tokens. Maintain `apiGroupPath`. |
| Existing bearer/refresh tokens | Define whether policy changes affect already-issued tokens. Initial MFA rollout should not claim revocation unless token invalidation is implemented. |
| Service authentication, scheduler, ECAs | Do not prompt for interactive OTP. Keep machine identities separate from human MFA-mandated accounts and document service credential controls. |
| Impersonation | Require suitable recent assurance from the actor, retain target semantics explicitly, and audit actor plus target. Do not treat impersonation as a way to evade a required factor. |
| Ecommerce and other plugin controllers | Propagate pending/success/error states through wrappers, after-login actions, and controller responses. Test each custom login route before enabling policy. |

## 7. Delivery phases

Each phase should be a reviewable change with the feature disabled by default until its entry paths are covered.

### Phase 0 — authentication boundary and regression contract

- Trace all callers of `userLogin`, `loginUserWithUserLoginId`, `doMainLogin`, and token issuance, including web `TOKEN` login, `checkJWTLogin`, `extensionConnectLogin`, and `generateAuthTokenService`.
- Preserve the existing `userLogin` contract and add `userLoginMfaPending` for primary credential validation without successful-login state, success history, or `userLoginSession` output. Reject service-auth use of this interactive path.
- Extract successful account-state updates into a private helper that the normal login path uses and a future MFA completion path can reuse.
- Keep the MFA-pending service unwired from browser and REST entry points until Phase 1 provides a server-side challenge and verified completion path. Do not create an unguarded finalization service.
- Regression-test normal login state updates, service authentication, and the deferred path's lack of successful-login state/history/session data.
- Do not add MFA screens or turn on group policy in this phase.

**Phase 0 result:** the `UserLoginTests` integration suite passes 4/4, including the new boundary tests. The MFA-pending service is not yet connected to `LoginWorker`, `checkLogin`, REST, or plugins, so there is no end-user MFA flow and no policy can be enabled yet. Phase 1 must add challenge-bound completion and browser integration before this service is used for login.

### Phase 1 — TOTP and recovery for standard web login

- Add factor, challenge, and audit model plus migration-compatible defaults.
- Add policy evaluator, minimum self-service TOTP enrollment/confirmation, one-time recovery codes, challenge expiry/binding, verification limits, and session finalization.
- Add common login/check-login/AJAX responses and challenge/enrollment views.
- Keep group-wide enforcement disabled while channel coverage is incomplete. Exercise the feature with test users first.

**Exit criteria:** password plus TOTP completes exactly one login; an enrolled factor is validated before it satisfies policy; wrong, expired, replayed, and over-limit codes do not authenticate; no normal session, final-success record, after-login event, or token exists before valid MFA; recovery code is consumed once.

**Phase 1 implementation and verification (2026-09-25):**

- Implemented TOTP enrollment and confirmation. An authenticator is not treated as active until the user proves possession of it.
- Added a session-bound login challenge. Successful-login state, login history, and the authenticated session are created only after valid MFA proof.
- Added one-time recovery codes and store only their hashes. Reuse is rejected.
- Added challenge expiry, replay rejection, and a five-attempt verification limit. Failed-code attempts update the counter and audit record; the browser flow treats them as failed challenges.
- Connected the standard browser login, check-login, and AJAX routes to the MFA flow; added challenge and enrollment screens and HTTPS/CSRF-protected enrollment events.
- Fixed unsupported `<iterate>` markup in the recovery screen by using `<iterate-section>`. The local HTTPS login page then rendered its normal title and username/password fields.
- Focused `UserLoginTests` passed 5/5 for deferred login state, service-auth rejection, enrollment/confirmation, challenge binding, pre-proof session/history checks, one-time finalization, replay, recovery-code handling, expiry, and lockout.
- A browser login-page render was checked, but authenticated enrollment, confirmation, challenge, and recovery interactions were not browser-tested.
- MFA is disabled by default. REST/token MFA and external-identity assurance are not Phase 1 deliverables; they remain deferred to later phases.

### Phase 2 — enrollment, administration, password change, and ecommerce

- Complete self-service enrollment/revocation, factor masking, backup-code rotation, and authorized administrator management.
- Apply the factor policy to forced and self-service password changes and define reset/recovery workflows.
- Update `applications/securityext` and the ecommerce controller wrappers/responses; inspect remaining plugin login overrides.
- Pilot group enforcement with a small privileged group, support/recovery process, and operational rollback instructions.

**Exit criteria:** enrollment cannot satisfy policy before validation; unauthorized users cannot view/change another user's factors; forced password change does not mutate the password before required proof; ecommerce and selected plugins cannot treat pending MFA as successful login.


**Phase 2 implementation and verification (2026-09-25):**

- Added `REQUIRE_AUTH_FACTOR` to security-group policy. A member of an enforced group without a validated factor is sent through forced TOTP enrolment; enrolment remains pending until possession is verified.
- Added self-service factor listing and revocation, recovery-code rotation, and metadata-only factor responses. Secret values are not returned by the listing operation.
- Added administrator factor and password-recovery actions protected by permission and a required reason. Successful password recovery records the administrator actor and reason in the audit event.
- Protected password changes in common security services, Party profile, ecommerce storefront, and ecommerce customer-profile flows. A password update is not called without recent factor proof.
- For MFA-required accounts, password hints and email-token recovery return a generic denial and create an audit event.
- Updated ecommerce `storeLogin` and `storeCheckLogin` to preserve MFA-required, forced-enrolment, password-change, and error outcomes rather than treating them as a successful store login. Forced-enrolment errors keep the provisioning URI only in the current server-side session and return the user to enrolment.
- `UserLoginTests` passed 9/9 against MySQL 8.4.11 with Connector/J 9.6.0. Regression checks cover forced enrolment, proof before password update, administrator permission/reason/audit, factor ownership, and ecommerce status propagation.
- The `securityext` wrapper unit test passed 1/1. `compileJava`, `compileTestGroovy`, `checkstyleTest`, `codenarcTest`, and `git diff --check` passed. Gradle 8.14.5 ran offline with unrelated Node setup tasks excluded because of the pre-existing broken npm symlink. The suite also exposed a challenge `PRIMARY_VERIFIED` marker regression, which was fixed.
- These results are service and controller-boundary checks. No real browser interaction was run for Phase 2.

**Known entry points outside Phase 2 coverage:**

- Janrain social login in `plugins/ecommerce` calls `LoginWorker.doBasicLogin` directly.
- Ecommerce account creation and email-based login-ID update also have direct `doBasicLogin` call sites.
- LDAP handlers call `userLogin` and then `doMainLogin` without the browser MFA gate.
- The Solr plugin wraps `LoginWorker.login`, but no controller mapping to that wrapper was found in this checkout.
- These routes are not claimed as MFA-protected. External identity and LDAP need Phase 4 policy work; Janrain and ecommerce self-service need a separate follow-up. REST/token MFA remains Phase 3 work.

**Rollout and recovery controls:**

- Keep MFA disabled and do not enable a required-authenticator group for production users until alternate routes are covered and supported browser flows are verified.
- Before a pilot, use an isolated MySQL test database/schema, rerun the focused integration suite, and verify the supported flows in a browser.
- Recover an MFA-required account through a security administrator with `SECURITY_PWD_UPDATE` permission and a reason of no more than 255 characters. Retain the actor/reason audit record. Password hints and email-token reset are intentionally blocked for these accounts.
- To disable browser MFA enforcement, set `security.login.authFactor.enable=false`. This does not delete factors or audit records.

### Phase 3 — email/SMS delivery and REST token protocol

- Add email delivery using configured Apache OFBiz email services; add SMS only with an explicit provider/adapter and operational configuration.
- Add a two-step REST challenge protocol around `/{apiGroupPath}/token`; issue tokens only after verified MFA; protect send/verify endpoints and keep public errors generic.
- Define refresh-token and already-issued-token policy and document client response handling.

**Exit criteria:** required-MFA REST callers receive no token before verification; challenge replay, cross-user, cross-tenant, expiry, and rate-limit cases fail; clients can distinguish MFA-required from ordinary authentication failure.

### Phase 4 — external assurance and remaining entry paths

- Implement policy adapters for configured SSO, JWT, certificate, reverse-proxy, and cross-webapp login-key flows.
- Accept upstream MFA only from explicitly trusted issuer/configuration data and verified assurance claims. Otherwise require local MFA or deny according to policy.
- Finish impersonation and service-account guidance, monitor audit events, and expand rollout beyond the pilot.

**Exit criteria:** every supported human authentication channel has an explicit policy and test; no in-scope path creates an authenticated session or token before required assurance; machine authentication remains compatible and isolated.

WebAuthn/passkeys, trusted-device cookies, action-specific step-up, and broad policy-management state machines are later work requiring separate scope review. Do not make them prerequisites for a secure TOTP/recovery release.

## 8. Required verification coverage

Tests should cover behavior and security invariants, not only helper methods:

- Policy truth table: feature disabled; no factor/no group policy; validated factor; pending factor; expired/revoked factor; active/expired group membership; system account; tenant isolation.
- TOTP known vectors, allowed time window, malformed input, and repeated code behavior as specified.
- Recovery code hash, one-time atomic consumption, rotation, and concurrent use.
- Challenge binding to pending session/user/delegator; wrong user/factor; expiry; logout/session rotation; tenant change; resend and verification rate limits; concurrent attempts.
- Login state invariants: no `session.userLogin`, final successful `UserLoginHistory`, authenticated cookie, REST JWT, or after-login event before required proof; exactly one final commit after proof.
- Login combinations: normal login; MFA-required login; forced enrollment; required password change; password reset/recovery; failed primary auth; failed MFA; impersonation.
- Alternate paths: header/remote user, X.509, `JWTManager.checkJWTLogin()`, web `TOKEN` login, `extensionConnectLogin()`, external login key, Basic REST token, `generateAuthTokenService`, refresh token, service authentication, scheduler/ECAs, and `loginUserWithUserLoginId` callers.
- Web/controller integration: common login, protected `checkLogin`, AJAX, ecommerce storefront wrappers, and all plugin login overrides found in the inventory.
- Entity/service authorization: direct generic entity access and remote service calls cannot expose or mutate secrets/factors outside authorized ownership and security administration.

## 9. Acceptance criteria for the feature

1. The default configuration preserves existing login behavior until an operator enables MFA.
2. An MFA-required interactive login cannot establish normal user session state or record a completed successful login before the second factor succeeds.
3. MFA-required password changes and recovery paths cannot be used to bypass the factor policy.
4. Only active, validated factors satisfy policy; factor ownership, tenant, session, challenge state, and expiry are checked server-side.
5. Secret material and one-time codes are not stored or logged in plaintext; destinations are masked in normal UI responses.
6. Verification and delivery attempts are bounded and handled safely under concurrent requests.
7. Each supported browser, plugin, REST, and external-login channel has an explicit policy and tests before group enforcement is enabled for it.
8. Service authentication and scheduled jobs remain compatible without accepting MFA-mandated human accounts as machine credentials.
9. Audit records distinguish primary authentication, factor challenge/verification, recovery, and completed login without containing secrets.
10. The feature can be rolled out to a pilot group with documented recovery and rollback procedures.

## 10. Decisions to record before implementation starts

- First release factor set: recommended TOTP plus recovery codes; decide whether verified email is in that release. Do not promise SMS until delivery support is selected.
- Scope of mandatory policy: identify which browser, plugin, REST, and external-auth paths are included before enabling the group flag.
- Upstream SSO/JWT assurance: define trusted issuer/configuration and which claims are accepted as proof of MFA.
- Enrollment rollout: decide whether existing users may enroll voluntarily before enforcement and how any grace period is represented.
- Recovery administration: define who can reset factors, what evidence/reason is required, notifications, and emergency access.
- Dependency choice and license/notice handling: verify the selected TOTP library against the current root Gradle dependency model and security review.
- REST response contract: choose status/body fields for MFA-required, challenge delivery, verification failure, and completed token issuance.
- Audit retention and operational monitoring: define event retention, access control, alerting thresholds, and privacy requirements.
