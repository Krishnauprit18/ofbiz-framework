# MFA Implementation: Simple File-by-File Guide

This guide explains how the current MFA changes in Apache OFBiz® fit together. It uses simple English and follows the request from login, through the MFA check, to the screens and tests.

## What is available now

- TOTP authenticator enrollment, login challenges, one-time recovery codes, factor listing and revocation, selected password-change protections, administrator recovery checks, and audit records are implemented in the local source.
- The global switch is off by default: `security.login.authFactor.enable=false` in `framework/security/config/security.properties`.
- The common login flow has service and integration-test coverage. The screens and controller routes exist, but a complete browser test of enrollment, challenge, and recovery has not been done.
- The ecommerce login wrapper passes MFA result states along, but the ecommerce controller does not currently define the MFA challenge/enrollment views and form routes. Do not treat ecommerce MFA login as ready for users.
- Janrain social login, ecommerce account creation/login-ID updates, LDAP login, and REST/token MFA are not fully covered by this implementation.

## 1. Data and service registration: what Apache OFBiz stores and can call

1. **`framework/security/entitydef/entitymodel.xml`** defines the records used by MFA:
   - `UserLoginAuthFactor` stores a user's authenticator or recovery-code record and its status.
   - `UserLoginAuthChallenge` stores a short-lived login challenge, its expiry, attempt count, and session binding.
   - `UserAuthenticationEvent` stores MFA and recovery audit events.
   - `SecurityGroup.requireAuthFactor` stores whether a security group requires an authenticator.

2. **`framework/security/servicedef/services.xml`** registers the MFA operations. The login and UI code calls these services to start enrollment, confirm a factor, list or revoke factors, create or verify a challenge, and generate recovery codes. They are internal services, not public REST APIs.

3. **`framework/security/ofbiz-component.xml`** loads that service definition file. Without this registration, the dispatcher would not know about the MFA services.

4. **`framework/security/src/main/java/org/apache/ofbiz/security/login/MfaServices.java`** contains the main factor and challenge rules:
   - It checks the global switch and the user's active security-group memberships.
   - It creates and confirms TOTP factors, and stores recovery-code hashes rather than reusable plain-text codes.
   - It returns factor metadata without returning secret values.
   - It expires and binds challenges, counts failed attempts, rejects replay, consumes recovery codes once, and writes audit events.
   - It checks ownership or administrator permission/reason before factor revocation.

## 2. Login: how the password check becomes an MFA challenge

1. **`framework/common/src/main/java/org/apache/ofbiz/common/login/LoginServices.java`** separates primary password validation from final successful-login processing. The pending-login service does not create normal successful-login state. The completion service checks the server-side pending challenge and only then finalizes login state.

2. **`framework/common/servicedef/services.xml`** registers the pending-login and final-login-completion services so the web login worker can call them.

3. **`framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/LoginWorker.java`** connects the services to web requests:
   - It checks the global setting and user's MFA policy after primary credentials are checked.
   - It sends the browser to a challenge or forced-enrollment result when needed.
   - It verifies the submitted code before finalizing login.
   - It records recent MFA proof for protected password changes and settings changes.

4. **`framework/security/config/security.properties`** controls the behavior. The current default is off. Other values set pending-challenge timeout, recent-proof window, maximum verification attempts, and TOTP issuer name.

## 3. Screens and browser routes: what the user sees

1. **`framework/common/webcommon/WEB-INF/common-controller.xml`** maps common browser requests to the login worker and MFA screens. It includes the login challenge, forced enrollment, ordinary enrollment, confirmation, and factor-revocation requests. The MFA-changing form requests require HTTPS and CSRF protection.

2. **`framework/common/widget/CommonScreens.xml`** describes what appears on screen:
   - `authFactorChallenge` asks for an authenticator code.
   - `authFactorForcedEnrollment` starts an enrollment required by policy.
   - `authFactorEnrollment` displays a QR code and secret key and asks for confirmation.
   - `authFactorSetup` provides self-service setup and the user's factor list.

3. **`framework/common/widget/CommonForms.xml`** defines the fields and form targets for those screens. It includes current-password checks for enrollment/revocation actions and separate factor lists for the account owner and security administration.

4. **`framework/common/config/SecurityextUiLabels.xml`** contains shared user-facing MFA messages, such as the message used when a factor is required before a password change.

5. **UI route examples:** after Apache OFBiz is running, the common screen is available at `/webtools/control/authFactorSetup` for an authenticated user. The normal login route is `/webtools/control/login`; a challenge is a response to login, not a separate page users should open directly. No self-service menu link was found, so setup currently requires the direct route.

## 4. Security administration: group policy, factor review, and recovery

1. **`framework/common/widget/SecurityForms.xml`** adds the Yes/No `requireAuthFactor` field to the security-group edit form. It also adds a reason field to the administrator password-recovery form.

2. **`framework/common/widget/SecurityScreens.xml`** places the security-group form on the group edit screen and places the administrator factor list on a user-login security screen. The factor list is shown only when the screen's security-permission condition passes.

   - Group policy UI: open `/webtools/control/FindSecurityGroup`, select a group, then edit it at `/webtools/control/EditSecurityGroup?groupId=GROUP_ID`.
   - User factor/admin password UI: open `/webtools/control/FindUserLogin`, select a user, then open `/webtools/control/editlogin?userLoginId=USER_LOGIN_ID`.

3. **`framework/common/webcommon/WEB-INF/security-controller.xml`** maps password update to the MFA-aware login worker. The security application routes also provide user and group administration screens.

4. **`framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/LoginWorker.java`** checks recent MFA proof for factor changes and password changes. For a protected administrative password reset, it checks permission and requires a reason before calling the existing password-update service.

5. **`framework/security/src/main/java/org/apache/ofbiz/security/login/MfaServices.java`** performs the final owner/admin checks and records the actor and reason in the audit record.

6. **Policy effect:** when the global switch is enabled, a user with an active factor is asked to use it at login. A user in a group marked as requiring MFA is required to enroll if they do not yet have a validated factor. Keep the switch off and do not enable group policy for production accounts until the uncovered login routes below are handled.

## 5. Password and recovery entry points

- **Common security password update:** `framework/common/webcommon/WEB-INF/security-controller.xml` maps `updatePassword` to `LoginWorker.updatePasswordWithMfa`.
- **Party profile password update:** `applications/party/webapp/partymgr/WEB-INF/controller.xml` maps `ProfileUpdatePassword` to the same MFA-aware handler.
- **Ecommerce password update:** `plugins/ecommerce/webapp/ecommerce/WEB-INF/controller.xml` maps the store password update to that handler. `plugins/ecommerce/minilang/customer/CustomerEvents.xml` also checks for recent MFA proof before its customer-profile password update.
- **Password hints and email-token recovery:** `applications/securityext/src/main/java/org/apache/ofbiz/securityext/login/LoginEvents.java` checks policy and blocks these recovery options for MFA-required accounts with a generic response and audit event.

## 6. Ecommerce login wrapper and current UI gap

- **`applications/securityext/src/main/java/org/apache/ofbiz/securityext/login/LoginEvents.java`** contains `storeLogin` and `storeCheckLogin`. They preserve MFA-required, forced-enrollment, password-change, and error results rather than treating every result as a successful store login.
- **`plugins/ecommerce/webapp/ecommerce/WEB-INF/controller.xml`** maps those result names for the store login requests and maps the password-change handler.
- **`applications/securityext/src/test/java/org/apache/ofbiz/securityext/login/LoginEventsTests.java`** checks that the wrapper preserves the MFA-related result states.
- **Current limitation:** the ecommerce controller has response names for MFA challenge/enrollment, but no matching MFA view maps or request maps for the common challenge/enrollment forms were found there. The wrapper test proves status propagation, not an ecommerce browser login. Complete this controller wiring and test it before using MFA on ecommerce logins.

## 7. Tests: what has been checked

- **`framework/common/src/test/groovy/org/apache/ofbiz/common/test/UserLoginTests.groovy`** contains service-level MySQL integration tests for deferred login completion, TOTP enrollment, challenge binding, recovery-code use, expiry/replay/attempt limits, factor ownership, enforced enrollment, password-change proof, and administrator recovery permission/reason/audit.
- **`applications/securityext/src/test/java/org/apache/ofbiz/securityext/login/LoginEventsTests.java`** checks ecommerce login result propagation.
- Recorded verification: the expanded `UserLoginTests` suite passed 9/9 on MySQL; the ecommerce wrapper test passed 1/1. Compile, test compilation, Checkstyle, CodeNarc, and whitespace checks also passed. These results do not replace browser testing.

## 8. Build and local database support files

These files support the local TOTP/MySQL build or environment; they do not add user-facing MFA screens:

- **`dependencies.gradle`** adds the TOTP library and MySQL JDBC driver to the build.
- **`gradle/libs.versions.toml`** records their versions and dependency coordinates.
- **`framework/entity/config/entityengine.xml`** changes local datasource group mappings from H2 to MySQL.
- **`framework/start/src/main/resources/org/apache/ofbiz/base/start/start.properties`** has a local admin-key change. Do not publish or copy its secret value.
- **`framework/rest-api/package-lock.json`** and **`themes/common-theme/webapp/common-theme/js/package-lock.json`** have dependency lockfile changes; they are not part of MFA request or screen behavior.

## 9. Boundaries before rollout

- The global MFA setting remains off by default.
- Browser testing of enrollment, login challenge, forced enrollment, recovery codes, factor revocation, and password recovery is still needed.
- Janrain social login, ecommerce account creation/login-ID update, LDAP, and REST/token authentication are not covered end to end.
- Do not enable production group enforcement until each login entry point available to those users has a tested MFA policy and response path.
