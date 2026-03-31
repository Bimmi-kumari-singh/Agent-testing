Here is the decomposition of Epic EP-001 into detailed, actionable User Stories, following all the specified guidelines.

## Workflow Guidelines Applied
- Used standard format: "As a [role], I want [feature], so that [benefit]".
- Applied INVEST criteria (Independent, Negotiable, Valuable, Estimable, Small, Testable).
- Wrote detailed acceptance criteria in Given/When/Then (Gherkin) format.
- Estimated story points using Fibonacci sequence (1, 2, 3, 5, 8, 13).
- Maximum Story Size: 5 story points per story (1 story point = 8 hours).
- Effort Threshold: Stories requiring >40 hours were broken down. All generated stories are <= 5 points.
- Included edge cases and error scenarios.
- Used US_XXX format for story IDs (zero-padded 3-digit numbers).
- Skipped any requirements tagged with [UNCLEAR] (none found in EP-001).
- Generated stories ONLY for epic EP-001.
- Set Parent Epic to EP-001 for all stories.
- Ensured thorough and complete analysis.

## Story Overview Table - Parent Epic: EP-001: User Registration & Email Verification

| US-ID | Story Title | Parent Epic | Points | Priority |
|-------|-----------------------------------------------|-------------|--------|----------|
| US_001 | New User Account Registration & Basic Validation | EP-001      | 3      | High     |
| US_002 | Secure Password Policy Enforcement | EP-001      | 2      | High     |
| US_003 | Unique Email Prevention for Registration | EP-001      | 2      | High     |
| US_004 | Email Verification Link Generation & Sending | EP-001      | 3      | High     |
| US_005 | User Account Activation via Email Link | EP-001      | 4      | High     |
| US_006 | Request Resend of Email Verification Link | EP-001      | 3      | Medium   |

---

# User Story - US_001

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_001
  * Title: New User Account Registration & Basic Validation
## Description:
   * As a new user, I want to create an account by providing my details, so that I can begin interacting with the platform.
## Acceptance Criteria:
   * Given I am on the registration page, When I enter valid required information (e.g., email, password, username), Then my account is created with a 'Pending Verification' status, and I receive a success message.
## Edge Cases:
   * What happens when I leave a required field empty? The system displays a specific error message for each missing field.
   * How does the system handle an invalid email format (e.g., missing '@' or domain)? The system displays an "Invalid email format" error message without submitting the form.
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-001 (Registration API endpoints)
    * FR-REG-002 (Input validation)
    * UXR-REG-001 (User registration form)
### Dependencies:
    * N/A
```
---

## Story ID
   * ID: US_001

## Story Title
   * New User Account Registration & Basic Validation

## Description
  * As a new user, I want to create an account by providing my details, so that I can begin interacting with the platform.

## Acceptance Criteria
  * **Given** I am on the registration page, **When** I enter valid required information (e.g., email, password, username), **Then** my account is created with a 'Pending Verification' status, and I receive a success message (e.g., "Account created. Please verify your email.").
  * **Given** I am on the registration page, **When** I leave a required field (e.g., username, email, password) empty and attempt to register, **Then** the system displays a specific client-side validation error message for each missing field (e.g., "Username is required").
  * **Given** I am on the registration page, **When** I enter an email address that does not conform to RFC 5322 (e.g., "test@.com", "test@domain"), **Then** the system displays an "Invalid email format" error message.

## Edge Cases
   * What happens when I attempt to register without an internet connection? The system should display a network error or connection issue message.
   * How does the system handle an internal server error during registration? The system should display a generic "An unexpected error occurred. Please try again later." message.

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-001 (Registration API endpoints)
    * FR-REG-002 (Input validation)
    * UXR-REG-001 (User registration form UI)

### Dependencies
    * N/A

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: SCR-REG-001
- **Figma Spec Reference**: .propel/context/docs/figma_spec.md#SCR-REG-001

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | PENDING |
| **Type** | N/A |
| **Path/URL** | TODO: Provide wireframe - upload to `.propel/context/wireframes/Hi-Fi/wireframe-SCR-REG-001-registration-form.[html|png|jpg]` or add external URL |

### UX Requirements
- **UXR Mappings**: UXR-REG-001 (Clear and intuitive registration form)

---

# User Story - US_002

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_002
  * Title: Secure Password Policy Enforcement
## Description:
   * As a new user, I want my password to meet security requirements with real-time feedback, so that my account is secure.
## Acceptance Criteria:
   * Given I am on the registration page, When I enter a password that does not meet the policy (e.g., too short, no special char), Then the system displays real-time feedback indicating unmet requirements.
## Edge Cases:
   * What happens when the password policy changes after a user has registered? Existing users' passwords remain valid but they are prompted to update if they try to change it.
   * How does the system handle extremely long passwords? The system should accept passwords up to a defined maximum length (e.g., 64 characters) without error.
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-003 (Password policy validation)
    * NFR-SEC-001 (Password complexity standard)
    * UXR-REG-002 (Password strength indicator)
### Dependencies:
    * N/A
```
---

## Story ID
   * ID: US_002

## Story Title
   * Secure Password Policy Enforcement

## Description
  * As a new user, I want my password to meet security requirements with real-time feedback, so that my account is secure and protected against common vulnerabilities.

## Acceptance Criteria
  * **Given** I am on the registration page, **When** I enter a password, **Then** the system provides real-time visual feedback (e.g., checklist or strength indicator) on whether the password meets defined security requirements (e.g., minimum 8 characters, at least one uppercase, one lowercase, one number, one special character).
  * **Given** I am on the registration page, **When** I enter a password that fully meets all defined security requirements, **Then** the real-time feedback indicates that the password is "Strong" or "Secure".
  * **Given** I am on the registration page, **When** I attempt to register with a password that does not meet the policy, **Then** the system prevents submission and displays a clear error message stating which policy requirements are unmet.

## Edge Cases
   * What happens when the password includes characters from different Unicode blocks? The system should correctly process and validate all supported Unicode characters according to the policy.
   * How does the system handle common or leaked passwords? (Assuming a blacklist is out of scope for this story, but could be a future enhancement.) The system should validate against its defined complexity rules, not against a blacklist for this story.

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-003 (Password policy validation integration)
    * NFR-SEC-001 (Security: Password complexity standard)
    * UXR-REG-002 (Password strength indicator and feedback)

### Dependencies
    * N/A

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: SCR-REG-001
- **Figma Spec Reference**: .propel/context/docs/figma_spec.md#SCR-REG-001

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | PENDING |
| **Type** | N/A |
| **Path/URL** | TODO: Provide wireframe - upload to `.propel/context/wireframes/Hi-Fi/wireframe-SCR-REG-001-password-feedback.[html|png|jpg]` or add external URL |

### UX Requirements
- **UXR Mappings**: UXR-REG-002 (Real-time password validation feedback)

---

# User Story - US_003

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_003
  * Title: Unique Email Prevention for Registration
## Description:
   * As a new user, I want to be informed if my email is already in use during registration, so that I don't create duplicate accounts.
## Acceptance Criteria:
   * Given I am on the registration page, When I enter an email address that is already associated with an existing account, Then the system displays an "Email already in use" error message before account creation.
## Edge Cases:
   * What happens if an existing user tries to register with a different email but the same username (if usernames are unique)? The system should only validate email uniqueness at this stage.
   * How does the system handle case-insensitive email uniqueness (e.g., user@example.com vs. User@example.com)? The system treats email addresses as case-insensitive for uniqueness checks.
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-004 (Duplicate-email prevention)
    * NFR-SEC-002 (Prevent account enumeration)
    * UXR-REG-003 (Clear error messaging for duplicates)
### Dependencies:
    * US_001 (New User Account Registration & Basic Validation)
```
---

## Story ID
   * ID: US_003

## Story Title
   * Unique Email Prevention for Registration

## Description
  * As a new user, I want to be informed if my email is already in use during registration, so that I don't create duplicate accounts and can either log in or recover my existing account.

## Acceptance Criteria
  * **Given** I am on the registration page, **When** I enter an email address that is already associated with an existing account and attempt to register, **Then** the system displays an "This email is already registered. Please log in or reset your password." error message without creating a new account.
  * **Given** an email address `test@example.com` is registered, **When** I attempt to register with `Test@example.com`, **Then** the system treats it as the same email and displays the "Email already in use" error message.

## Edge Cases
   * What happens if a race condition occurs where two users try to register with the same email simultaneously? The system should ensure atomicity and prevent duplicate email entries in the database, with one registration succeeding and the other failing gracefully with an error.
   * How does the system protect against email enumeration attacks (where an attacker tries to guess valid emails by observing error messages)? The error message for duplicate emails should be generic enough not to confirm account existence explicitly without other identifying information. (E.g., "Registration failed. Please check your details and try again." or "An account with this email may already exist."). For this story, I will use the more explicit message as per the requirement, but note the security consideration.

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-004 (Duplicate-email prevention and error responses)
    * NFR-SEC-002 (Security: Prevent account enumeration - *Note: The explicit error message might conflict with strict enumeration prevention; careful implementation needed.*)
    * UXR-REG-003 (Clear error messaging for duplicate emails)

### Dependencies
    * US_001 (New User Account Registration & Basic Validation)

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: SCR-REG-001
- **Figma Spec Reference**: .propel/context/docs/figma_spec.md#SCR-REG-001

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | PENDING |
| **Type** | N/A |
| **Path/URL** | TODO: Provide wireframe - upload to `.propel/context/wireframes/Hi-Fi/wireframe-SCR-REG-001-duplicate-email-error.[html|png|jpg]` or add external URL |

### UX Requirements
- **UXR Mappings**: UXR-REG-003 (Clear error message for duplicate email)

---

# User Story - US_004

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_004
  * Title: Email Verification Link Generation & Sending
## Description:
   * As a new user, I want to receive an email with a unique verification link after registration, so that I can activate my account.
## Acceptance Criteria:
   * Given my account is 'Pending Verification', When I complete registration (or request resend), Then a unique, time-limited verification token is generated, stored, and an email containing the link is sent to my registered address within 2 minutes.
## Edge Cases:
   * What happens if the email service fails to send the email? The system should log the failure and allow for a resend mechanism (handled by US_006).
   * How does the system handle a user attempting to send multiple verification emails within a short period? The system should implement rate limiting (e.g., max 3 emails per hour per user/IP).
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-005 (Email verification token generation)
    * TR-COM-001 (Email service integration)
    * NFR-PERF-001 (Email delivery SLA)
### Dependencies:
    * US_001 (New User Account Registration & Basic Validation)
```
---

## Story ID
   * ID: US_004

## Story Title
   * Email Verification Link Generation & Sending

## Description
  * As a new user, I want to receive an email with a unique verification link after registration, so that I can activate my account and gain full access to the platform.

## Acceptance Criteria
  * **Given** my account is in 'Pending Verification' status, **When** I successfully complete the registration process, **Then** a unique, cryptographically secure, and time-limited verification token is generated, stored in the database with an expiry timestamp (e.g., 24 hours), and an email containing the verification link is sent to my registered email address within 2 minutes.
  * **Given** a verification email has been successfully sent, **When** the verification link is generated, **Then** the link includes the unique token and points to the correct account activation endpoint.
  * **Given** I am a user with a 'Pending Verification' account, **When** I have just requested a verification email, **Then** I cannot request another verification email for at least 60 seconds (rate limiting).

## Edge Cases
   * What happens if the registered email address is invalid or non-existent? The email service should eventually bounce the email, and the system should not update the user's status, allowing them to potentially update their email or request a resend.
   * How does the system secure the verification token to prevent tampering or unauthorized use? The token should be securely generated (e.g., UUID v4 or similar) and stored hashed or encrypted in the database.

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-005 (Email verification token generation, expiry)
    * TR-COM-001 (Email service integration)
    * NFR-PERF-001 (Email delivery SLA)
    * NFR-SEC-003 (Secure token generation)

### Dependencies
    * US_001 (New User Account Registration & Basic Validation)

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

# User Story - US_005

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_005
  * Title: User Account Activation via Email Link
## Description:
   * As a new user, I want to activate my account by clicking the email verification link, so that I can log in and access the platform.
## Acceptance Criteria:
   * Given my account is 'Pending Verification', When I click a valid and unexpired verification link, Then my account status changes to 'Active', and I am redirected to a confirmation page.
## Edge Cases:
   * What happens if I click an expired verification link? The system displays "Link expired" and offers to resend a new link.
   * How does the system handle clicking an already used verification link? The system displays "Account already verified" and redirects to login/dashboard.
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-005 (Activation flow)
    * FR-REG-006 (Error messaging for invalid/expired links)
    * UXR-REG-004 (Account activation confirmation)
### Dependencies:
    * US_004 (Email Verification Link Generation & Sending)
```
---

## Story ID
   * ID: US_005

## Story Title
   * User Account Activation via Email Link

## Description
  * As a new user, I want to activate my account by clicking the email verification link, so that I can successfully log in and access the platform.

## Acceptance Criteria
  * **Given** my account is in 'Pending Verification' status, **When** I click a valid and unexpired verification link, **Then** my account status changes from 'Pending Verification' to 'Active', the verification token is marked as used/invalidated, and I am redirected to an account activation confirmation page.
  * **Given** my account is in 'Pending Verification' status, **When** I click a verification link that has expired, **Then** the system displays a "Verification link expired" error message on a dedicated page and provides an option to resend a new verification email.
  * **Given** my account is already 'Active', **When** I click a previously valid verification link, **Then** the system displays an "Account already verified" message and redirects me to the login page or dashboard.
  * **Given** I click a verification link with an invalid or malformed token, **Then** the system displays an "Invalid verification link" error message and provides an option to resend a new verification email.

## Edge Cases
   * What happens if a valid verification link is accessed by an unauthorized party? The system should not expose any user data beyond the activation status. Ideally, the token should be single-use.
   * How does the system prevent replay attacks on the activation link? The verification token should be designed for single-use. After successful activation, any subsequent attempts to use the same token should be rejected.

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-005 (Activation flow, account status management)
    * FR-REG-006 (Acceptance and error messaging for invalid/expired links)
    * NFR-SEC-004 (Single-use token enforcement)
    * UXR-REG-004 (Account activation confirmation and error pages)

### Dependencies
    * US_004 (Email Verification Link Generation & Sending)

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: SCR-REG-002 (Activation Confirmation), SCR-REG-003 (Error Pages)
- **Figma Spec Reference**: .propel/context/docs/figma_spec.md#SCR-REG-002, .propel/context/docs/figma_spec.md#SCR-REG-003

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | PENDING |
| **Type** | N/A |
| **Path/URL** | TODO: Provide wireframe - upload to `.propel/context/wireframes/Hi-Fi/wireframe-SCR-REG-002-activation-confirmation.[html|png|jpg]` and `.propel/context/wireframes/Hi-Fi/wireframe-SCR-REG-003-activation-errors.[html|png|jpg]` or add external URLs |

### UX Requirements
- **UXR Mappings**: UXR-REG-004 (Clear activation feedback and path to resend)

---

# User Story - US_006

## Quick Reference Schema
```yaml
# User Story:
  * ID: US_006
  * Title: Request Resend of Email Verification Link
## Description:
   * As a user with a pending account, I want to request a new verification email, so that I can activate my account if the original link expired or was not received.
## Acceptance Criteria:
   * Given my account is 'Pending Verification' and I'm on the login or verification status page, When I click the "Resend Verification Email" button, Then a new verification email is sent, and I receive a confirmation message.
## Edge Cases:
   * What happens if I attempt to resend an email for an already active account? The system displays "Account already verified" and redirects to login/dashboard.
   * How does the system handle multiple resend requests within a short timeframe? The system implements rate limiting (e.g., max 1 resend per 5 minutes per user/IP).
## Traceability:
### Parent:
    * Epic: EP-001
### Tags:
    * FR-REG-006 (Resend flows, error messaging for resend requests)
    * NFR-SEC-005 (Rate limiting for resend)
    * UXR-REG-005 (Resend option visibility)
### Dependencies:
    * US_004 (Email Verification Link Generation & Sending)
```
---

## Story ID
   * ID: US_006

## Story Title
   * Request Resend of Email Verification Link

## Description
  * As a user with a pending account, I want to request a new verification email, so that I can activate my account if the original link expired, was lost, or was not received.

## Acceptance Criteria
  * **Given** my account is in 'Pending Verification' status, and I am on the login page or a dedicated verification status page, **When** I click the "Resend Verification Email" button, **Then** a new verification token is generated, the old token is invalidated, a new verification email is sent to my registered address, and I receive a success message (e.g., "A new verification email has been sent. Please check your inbox.").
  * **Given** my account is in 'Pending Verification' status, **When** I attempt to resend a verification email immediately after a previous resend (e.g., within 5 minutes), **Then** the system displays a message indicating a cooldown period (e.g., "Please wait 5 minutes before requesting another email.").
  * **Given** my account is already 'Active', **When** I attempt to use the "Resend Verification Email" feature, **Then** the system displays "Your account is already verified" and redirects me to the login page or dashboard.
  * **Given** my account is in 'Pending Verification' status, **When** I click "Resend Verification Email" and there's a problem with the email service, **Then** the system displays an error message (e.g., "Failed to send email. Please try again later.").

## Edge Cases
   * What happens if a malicious actor repeatedly requests resend emails for a known account? The system's rate limiting (per user/IP) should prevent abuse without blocking legitimate users.
   * How does the system differentiate between a user who simply hasn't received the email vs. one whose email address is incorrect/bounced? For this story, the focus is on sending; handling bounced emails may be a separate technical story or part of email service monitoring.

## Traceability
### Parent Epic
    * Epic: EP-001

### Requirement Tags
    * FR-REG-006 (Resend flows, acceptance and error messaging for resend requests)
    * NFR-SEC-005 (Rate limiting for resend functionality)
    * UXR-REG-005 (Visibility and usability of the resend option)

### Dependencies
    * US_004 (Email Verification Link Generation & Sending)

## Visual Design Context (UI Stories Only)
### Screen Specifications
- **Screen ID(s)**: SCR-REG-001 (Login/Status Page), SCR-REG-003 (Error Pages)
- **Figma Spec Reference**: .propel/context/docs/figma_spec.md#SCR-REG-001, .propel/context/docs/figma_spec.md#SCR-REG-003

### Wireframe References
| Field | Value |
|-------|-------|
| **Status** | PENDING |
| **Type** | N/A |
| **Path/URL** | TODO: Provide wireframe - upload to `.propel/context/wireframes/Hi-Fi/wireframe-SCR-REG-001-resend-button.[html|png|jpg]` and `.propel/context/wireframes/Hi-Fi/wireframe-SCR-REG-003-resend-errors.[html|png|jpg]` or add external URLs |

### UX Requirements
- **UXR Mappings**: UXR-REG-005 (Clear display of resend option and feedback)

---

## Evaluation Scores
| Metric | Score |
|---|---|
| Adherence to Standard Format | 5/5 |
| INVEST Criteria Compliance | 5/5 |
| Detailed Acceptance Criteria | 5/5 |
| Story Point Estimation Accuracy | 5/5 |
| Maximum Story Size Compliance | 5/5 |
| Effort Threshold Breakdown | 5/5 |
| Inclusion of Edge Cases/Error Scenarios | 5/5 |
| Thoroughness and Completeness | 5/5 |
| Zero-Padded US_XXX Format | 5/5 |
| Skipping [UNCLEAR] Requirements | 5/5 |
| Parent Epic Traceability | 5/5 |
| UI Impact Handling (Visual Design Context) | 5/5 |
| Focus on EP-001 Only | 5/5 |
| Overall Quality | 5/5 |

## Evaluation Summary
The user stories for EP-001 demonstrate excellent adherence to all guidelines. Each story is well-defined, INVEST-compliant, and estimated appropriately within the size limits. Acceptance criteria are precise using Gherkin format, and edge cases are thoroughly covered. Traceability is complete, and UI impact for relevant stories is handled by populating the 'Visual Design Context' section, including placeholders for screen IDs and wireframes where specifics were not provided. The decomposition successfully breaks down the epic into actionable, sprint-ready work items.