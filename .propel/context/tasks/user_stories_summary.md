Here are the detailed User Stories for Epic EP-001, following all the specified guidelines.

## Rules Used by the Workflow
- `rules/ai-assistant-usage-policy.md`: Prioritized explicit user commands, minimal output, no decorative elements.
- `rules/code-anti-patterns.md`: Implicitly considered for good story design (e.g., preventing God objects).
- `rules/dry-principle-guidelines.md`: Ensured no redundant information and focused on generating new content.
- `rules/iterative-development-guide.md`: Followed the phased workflow of outlining stories first, then detailing each.
- `rules/language-agnostic-standards.md`: Applied principles of KISS, YAGNI, and clear naming for story descriptions.
- `rules/markdown-styleguide.md`: Ensured strict adherence to Markdown formatting, headings, lists, and code blocks.
- `rules/performance-best-practices.md`: Implicitly considered when defining efficient system responses and rate limiting.
- `rules/security-standards-owasp.md`: Incorporated security considerations like password policy, uniqueness checks, token generation/expiry.
- `rules/software-architecture-patterns.md`: Implicitly considered in decomposing functionalities to avoid architectural anti-patterns.

## Story Overview Table

| US-ID  | Story Title                      | Parent Epic | Points | Priority |
| :----- | :------------------------------- | :---------- | :----- | :------- |
| US_001 | User Account Registration        | EP-001      | 5      | High     |
| US_002 | Send Account Verification Email  | EP-001      | 3      | High     |
| US_003 | Activate Account via Email Link  | EP-001      | 4      | High     |
| US_004 | Resend Account Verification Email | EP-001      | 4      | Medium   |
| US_005 | Handle Invalid Verification Links | EP-001      | 3      | Medium   |

---

# User Story - US_001

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_001
  * Title: User Account Registration
## Description:
   * As a new user, I want to register for an account with my email and password, so that I can start using the platform.
## Acceptance Criteria:
   * Given I am on the registration page, When I enter a valid, unique email and a password conforming to policy, Then my account is created with "Pending Verification" status and a verification email is triggered.
## Edge Cases:
   * What happens when I try to register with an already registered email address?
   * How does the system handle an email address that does not conform to RFC 5322?
   * What happens when my password does not meet the defined security policy?
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-001 (Registration), FR-REG-002 (Client-side validation), FR-REG-003 (Server-side validation), FR-REG-004 (Duplicate-email prevention), NFR-SEC-001 (Password policy), NFR-VAL-001 (Email format validation)
### Dependencies:
    * US_002 (Send Account Verification Email)
```
---

## Story ID
   * ID: US_001

## Story Title
   * User Account Registration

## Description
  * As a new user, I want to register for an account with my email and password, so that I can start using the platform.

## Acceptance Criteria
  * **Given** I am on the registration page, **When** I enter a valid, unique email address and a password conforming to the defined security policy, **Then** my account is successfully created with a "Pending Verification" status, and a verification email dispatch is triggered.
  * **Given** I am on the registration page, **When** I enter an email address that is already registered, **Then** the system displays an error message "This email address is already in use."
  * **Given** I am on the registration page, **When** I enter an email address that is not in a valid RFC 5322 format, **Then** the system displays an error message "Please enter a valid email address format."
  * **Given** I am on the registration page, **When** I enter a password that does not meet the defined security policy (e.g., insufficient length, missing character types), **Then** the system displays real-time feedback and an error message "Password does not meet requirements."

## Edge Cases
   * What happens when a user attempts to submit the form without completing all required fields? (System provides specific field validation errors)
   * How does the system handle multiple concurrent registration attempts from the same IP address within a short period? (Rate-limit registration attempts to prevent spam)
   * What happens if the email verification trigger fails after account creation but before email dispatch? (Account remains in "Pending Verification" and system logs error for retry/admin action)

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-001 (Registration API endpoints), FR-REG-002 (Client-side validation), FR-REG-003 (Server-side validation), FR-REG-004 (Duplicate-email prevention), NFR-SEC-001 (Password policy), NFR-VAL-001 (Email format validation)

### Dependencies
    * US_002 (Send Account Verification Email)

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: N/A
- **Figma Spec Reference**: N/A

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | PENDING |
| **Type** | N/A |
| **Path/URL** | TODO: Provide wireframe - upload to `.propel/context/wireframes/Hi-Fi/wireframe-SCR-001-registration-form.html` or add external URL |

### UX Requirements
- **UXR Mappings**: N/A

---

# User Story - US_002

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_002
  * Title: Send Account Verification Email
## Description:
   * As the system, I want to send an email verification link to new users, so that their email ownership can be confirmed and their account activated.
## Acceptance Criteria:
   * Given a new user account is created with "Pending Verification" status, When the system receives the trigger to send a verification email, Then a unique, time-limited verification token is generated, stored, and an email containing the activation link is sent to the user's registered email address.
## Edge Cases:
   * What happens if the email service is temporarily unavailable?
   * How does the system handle the generation of a token for an email address that is already active?
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-005 (Email verification token generation, email delivery integration), NFR-SEC-002 (Token uniqueness/security), NFR-OPS-001 (Email service reliability)
### Dependencies:
    * N/A
```
---

## Story ID
   * ID: US_002

## Story Title
   * Send Account Verification Email

## Description
  * As the system, I want to send an email verification link to new users, so that their email ownership can be confirmed and their account activated.

## Acceptance Criteria
  * **Given** a new user account is created with "Pending Verification" status, **When** the system receives the trigger to send a verification email, **Then** a unique, cryptographically secure, and time-limited verification token is generated, securely associated with the user's account, and an email containing the activation link (with the token) is successfully dispatched to the user's registered email address.
  * **Given** a verification token is generated, **When** the system sends the email, **Then** the verification token is marked with an expiration timestamp, typically 24 hours from generation.
  * **Given** a user account is already active, **When** an attempt is made to send a verification email for that account, **Then** the system does not generate a new token or send an email, and logs the attempt.

## Edge Cases
   * What happens if the email delivery service (e.g., SendGrid, Mailgun) is temporarily unavailable or returns an error? (System logs the error, marks the email as undelivered, and may trigger an automated retry mechanism or alert an administrator.)
   * How does the system ensure the uniqueness of the generated verification token? (Utilizes a UUID or similar robust generation mechanism with uniqueness checks upon storage.)
   * What happens if the email address is invalid or bounces? (Email service should notify the system, which logs the bounce and potentially updates the user account status to reflect an undeliverable email.)

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-005 (Email verification token generation, email delivery integration), NFR-SEC-002 (Token uniqueness/security), NFR-OPS-001 (Email service reliability)

### Dependencies
    * N/A

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: N/A
- **Figma Spec Reference**: N/A

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | N/A |
| **Type** | N/A |
| **Path/URL** | N/A |

### UX Requirements
- **UXR Mappings**: N/A

---

# User Story - US_003

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_003
  * Title: Activate Account via Email Link
## Description:
   * As a new user, I want to activate my account by clicking the link in the verification email, so that I can gain full access to the platform.
## Acceptance Criteria:
   * Given I have received a valid, non-expired verification link, When I click the link, Then my account status is updated to "Active", and I am redirected to a confirmation page or login page.
## Edge Cases:
   * What happens if I use an expired verification link?
   * How does the system handle a verification link that has already been used?
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-006 (Activation flow), UXR-CONF-001 (Confirmation/redirect)
### Dependencies:
    * US_002 (Send Account Verification Email)
```
---

## Story ID
   * ID: US_003

## Story Title
   * Activate Account via Email Link

## Description
  * As a new user, I want to activate my account by clicking the link in the verification email, so that I can gain full access to the platform.

## Acceptance Criteria
  * **Given** I have received a valid, non-expired verification link in my email, **When** I click the link, **Then** the system validates the token, updates my account status from "Pending Verification" to "Active", and redirects me to a successful account activation confirmation page.
  * **Given** my account has been successfully activated via a verification link, **When** I attempt to use the *same* verification link again, **Then** the system displays a message indicating the account is already active and offers to redirect to the login page.
  * **Given** I am redirected to the account activation confirmation page, **When** the page loads, **Then** it clearly states that my account is active and provides an option to proceed to the login page.

## Edge Cases
   * What happens if the verification link's token is malformed or invalid? (System displays an error message "Invalid verification link" and offers to resend a new link.)
   * How does the system prevent unauthorized account activation attempts using guessed or brute-forced tokens? (Tokens are sufficiently long and complex, and attempts to use invalid tokens are logged and rate-limited.)

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-006 (Activation flow, account status management), NFR-SEC-003 (Token validation), UXR-CONF-001 (Confirmation/redirect)

### Dependencies
    * US_002 (Send Account Verification Email)

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: N/A
- **Figma Spec Reference**: N/A

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | PENDING |
| **Type** | N/A |
| **Path/URL** | TODO: Provide wireframe - upload to `.propel/context/wireframes/Hi-Fi/wireframe-SCR-002-activation-confirm.html` or add external URL |

### UX Requirements
- **UXR Mappings**: N/A

---

# User Story - US_004

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_004
  * Title: Resend Account Verification Email
## Description:
   * As a new user with a pending account, I want to request a new verification email, so that I can activate my account if the original link was lost or expired.
## Acceptance Criteria:
   * Given I have a "Pending Verification" account, When I request to resend the verification email (e.g., from login/registration page), Then the system generates a new verification token, invalidates any previous tokens for that user, and sends a new verification email.
## Edge Cases:
   * What happens if I request multiple resends within a short timeframe?
   * How does the system handle a resend request for an already active account?
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-005 (Resend flows), UXR-RESEND-001 (Resend button/link), NFR-SEC-004 (Rate limiting)
### Dependencies:
    * US_002 (Send Account Verification Email)
```
---

## Story ID
   * ID: US_004

## Story Title
   * Resend Account Verification Email

## Description
  * As a new user with a pending account, I want to request a new verification email, so that I can activate my account if the original link was lost or expired.

## Acceptance Criteria
  * **Given** I am a user with a "Pending Verification" account status, and I am on a page (e.g., login or resend page) that offers to resend the verification email, **When** I provide my registered email address and click the "Resend Verification Email" button, **Then** the system generates a new verification token, invalidates any previously issued tokens for my account, and sends a new email containing the updated activation link.
  * **Given** I have a "Pending Verification" account, **When** I request to resend a verification email, **Then** the system implements a rate-limiting mechanism (e.g., max 3 requests per hour per email address) and displays an appropriate message if exceeded.
  * **Given** I request to resend a verification email for an email address associated with an *already active* account, **When** I click the resend button, **Then** the system displays a message informing me that my account is already active and offers to redirect to the login page, without sending a new email.

## Edge Cases
   * What happens if the system attempts to resend an email for an account that no longer exists? (System gracefully handles the non-existent account, logs the attempt, and informs the user if necessary (e.g., "Email not found").)
   * How does the system ensure the "old" verification links become invalid immediately upon generating a new one? (The system updates the user's record with the new token and invalidates the old one, possibly by setting an "used" or "revoked" flag or overwriting.)

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-005 (Resend flows, token regeneration), UXR-RESEND-001 (Resend button/link), NFR-SEC-004 (Rate limiting), NFR-AVAIL-001 (System responsiveness)

### Dependencies
    * US_002 (Send Account Verification Email)

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: N/A
- **Figma Spec Reference**: N/A

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | PENDING |
| **Type** | N/A |
| **Path/URL** | TODO: Provide wireframe - upload to `.propel/context/wireframes/Hi-Fi/wireframe-SCR-003-resend-prompt.html` or add external URL |

### UX Requirements
- **UXR Mappings**: N/A

---

# User Story - US_005

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_005
  * Title: Handle Invalid Verification Links
## Description:
   * As a user, I want clear feedback when trying to use an invalid or expired verification link, so that I understand what went wrong and how to proceed.
## Acceptance Criteria:
   * Given I click a verification link with an expired token, When the system processes the request, Then it displays an error message "Verification link expired. Please request a new one." and offers a resend option.
## Edge Cases:
   * What happens if the verification link contains a token for a non-existent user?
   * How does the system distinguish between an expired token and a malformed token?
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-006 (Error messaging for links), UXR-ERROR-001 (Clear error messages), NFR-SEC-005 (Token validation errors)
### Dependencies:
    * US_003 (Activate Account via Email Link), US_004 (Resend Account Verification Email)
```
---

## Story ID
   * ID: US_005

## Story Title
   * Handle Invalid Verification Links

## Description
  * As a user, I want clear feedback when trying to use an invalid or expired verification link, so that I understand what went wrong and how to proceed.

## Acceptance Criteria
  * **Given** I click a verification link where the associated token has expired, **When** the system attempts to validate the token, **Then** it displays an error page with the message "Verification link expired. Please request a new one." and provides a prominent link or button to initiate the resend verification email process.
  * **Given** I click a verification link where the token is malformed (e.g., incorrect format, invalid characters) or does not exist in the system, **When** the system attempts to validate the token, **Then** it displays an error page with the message "Invalid verification link. Please check the link or request a new one." and offers a resend option.
  * **Given** an error occurs during the verification link processing, **When** the error page is displayed, **Then** it should not expose any sensitive system information, but provide sufficient context for the user and an easy way to recover.

## Edge Cases
   * What happens if the user tries to activate an account with a link that belongs to a user whose account has been deactivated or deleted? (System should indicate that the account cannot be activated and suggest contacting support or creating a new account.)
   * How does the system handle potential brute-force attempts on verification links with an invalid token? (The system logs invalid token attempts, and repeated attempts from the same IP address or user agent should be rate-limited and potentially blocked.)

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-006 (Error messaging for invalid/expired links), UXR-ERROR-001 (Clear error messages), NFR-SEC-005 (Token validation errors), NFR-LOG-001 (Error logging)

### Dependencies
    * US_003 (Activate Account via Email Link), US_004 (Resend Account Verification Email)

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: N/A
- **Figma Spec Reference**: N/A

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | PENDING |
| **Type** | N/A |
| **Path/URL** | TODO: Provide wireframe - upload to `.propel/context/wireframes/Hi-Fi/wireframe-SCR-004-invalid-link-error.html` or add external URL |

### UX Requirements
- **UXR Mappings**: N/A

---

## Evaluation Scores

| Criteria               | US_001 | US_002 | US_003 | US_004 | US_005 | Average Score |
| :--------------------- | :----- | :----- | :----- | :----- | :----- | :------------ |
| **INVEST (Independence)** | 5      | 5      | 4      | 4      | 5      | 4.6           |
| **INVEST (Negotiable)**  | 5      | 5      | 5      | 5      | 5      | 5.0           |
| **INVEST (Valuable)**    | 5      | 5      | 5      | 4      | 4      | 4.6           |
| **INVEST (Estimable)**   | 5      | 5      | 5      | 5      | 5      | 5.0           |
| **INVEST (Small)**       | 5      | 5      | 5      | 5      | 5      | 5.0           |
| **INVEST (Testable)**    | 5      | 5      | 5      | 5      | 5      | 5.0           |
| **Gherkin Format**     | 5      | 5      | 5      | 5      | 5      | 5.0           |
| **Edge Cases Coverage**| 5      | 4      | 4      | 4      | 5      | 4.4           |
| **Traceability**       | 5      | 5      | 5      | 5      | 5      | 5.0           |
| **Story Point Adherence** | 5      | 5      | 5      | 5      | 5      | 5.0           |
| **UI Context Accuracy**| 4      | 5      | 4      | 4      | 4      | 4.2           |
| **Overall Score**      | 4.8    | 4.8    | 4.7    | 4.6    | 4.7    | **4.72**      |

## Evaluation Summary
The user stories for EP-001 demonstrate high quality, adhering strictly to INVEST principles and Gherkin format. All stories are appropriately sized within the 5-point limit. Traceability is complete, and edge cases are well-covered, ensuring thoroughness. UI context is accurately represented as PENDING, awaiting further design input. Minor areas for improvement exist in slightly expanding edge cases for US_002-US_004 and refining UI context for pending wireframes. Overall, the decomposition is comprehensive and actionable.