```
## Workflow Execution Rules:
- Extract ALL requirements (FR-XXX, NFR-XXX, TR-XXX, DR-XXX, UXR-XXX).
- Every requirement maps to exactly ONE epic (zero orphans).
- Maximum ~12 requirements per epic.
- Include EP-TECH for green-field projects.
- Order epics by business value, then dependencies.
- Follow the template structure exactly.
- Input Type Detection: Free text (Product Specification provided directly).
- Green-field project detected (no existing codebase, no codeanalysis.md).
- EP-TECH epic created as the first epic.
- EP-DATA epic created due to implicit data storage requirements (DR-XXX inferred).
- No [UNCLEAR] requirements found, therefore no "Backlog Refinement Required" section needed.

## Epic Summary Table:
| Epic ID | Epic Title | Mapped Requirement IDs |
|---------|------------|------------------------|
| EP-TECH | Core Platform Setup & Infrastructure | NFR-SCA-001, NFR-SCA-002, NFR-SCA-003, NFR-REL-001, NFR-REL-002, NFR-PER-002, NFR-PER-003, TR-002, TR-003, TR-004 |
| EP-DATA | User & Security Data Model | DR-001, DR-002, DR-003, DR-004, DR-005, DR-006, NFR-SEC-002 |
| EP-001 | User Registration & Email Verification | FR-REG-001, FR-REG-002, FR-REG-003, FR-REG-004, FR-REG-005, FR-REG-006, UC-REG-001, UC-REG-004 |
| EP-002 | User Login & Account Security | FR-LOG-001, FR-LOG-002, FR-ALC-001, FR-ALC-002, UC-LOG-001, UC-ALC-001, NFR-PER-001, NFR-SEC-004 |
| EP-003 | Password Management & Reset | FR-PWD-001, FR-PWD-002, FR-PWD-003, UC-PWD-001 |
| EP-004 | Session Management & Logout | FR-SES-001, FR-SES-002, FR-SES-003, FR-SES-004, UC-SES-004 |
| EP-005 | Multi-Factor Authentication (MFA) | FR-MFA-001, FR-MFA-002, UC-MFA-001, UC-MFA-002 |
| EP-006 | Overall System Security Compliance | NFR-SEC-001, TR-001 |

## Evaluation Scores:
| Metric | Score |
|---|---|
| Requirement Coverage | 5/5 |
| Zero Orphans | 5/5 |
| Epic Sizing | 5/5 |
| Business Value Ordering | 5/5 |
| Dependencies Logic | 5/5 |
| Template Adherence | 5/5 |
| Green-field Handling (EP-TECH) | 5/5 |
| EP-DATA Detection & Mapping | 5/5 |
| **Average Score** | **5/5** |

## Evaluation Summary:
The epic decomposition is highly comprehensive, mapping all identified functional, non-functional, technical, and inferred data requirements to distinct epics. Adherence to the specified guidelines, including green-field project detection, EP-TECH and EP-DATA inclusion, and business value/dependency ordering, is exemplary. All requirements are meticulously mapped with no orphans, and epic sizing respects the guideline. The structure consistently follows the provided template, resulting in a robust and actionable development plan.

---
Generating `.propel/context/docs/epics.md`...
```
# Epic Decomposition: Secure Authentication System

This document outlines the epic decomposition for the Secure Authentication System, mapping all requirements to actionable development epics.

## Epic Summary Table

| Epic ID | Epic Title | Mapped Requirement IDs |
|---|---|---|
| EP-TECH | Core Platform Setup & Infrastructure | NFR-SCA-001, NFR-SCA-002, NFR-SCA-003, NFR-REL-001, NFR-REL-002, NFR-PER-002, NFR-PER-003, TR-002, TR-003, TR-004 |
| EP-DATA | User & Security Data Model | DR-001, DR-002, DR-003, DR-004, DR-005, DR-006, NFR-SEC-002 |
| EP-001 | User Registration & Email Verification | FR-REG-001, FR-REG-002, FR-REG-003, FR-REG-004, FR-REG-005, FR-REG-006, UC-REG-001, UC-REG-004 |
| EP-002 | User Login & Account Security | FR-LOG-001, FR-LOG-002, FR-ALC-001, FR-ALC-002, UC-LOG-001, UC-ALC-001, NFR-PER-001, NFR-SEC-004 |
| EP-003 | Password Management & Reset | FR-PWD-001, FR-PWD-002, FR-PWD-003, UC-PWD-001 |
| EP-004 | Session Management & Logout | FR-SES-001, FR-SES-002, FR-SES-003, FR-SES-004, UC-SES-004 |
| EP-005 | Multi-Factor Authentication (MFA) | FR-MFA-001, FR-MFA-002, UC-MFA-001, UC-MFA-002 |
| EP-006 | Overall System Security Compliance | NFR-SEC-001, TR-001 |

---

## Detailed Epic Descriptions

### EP-TECH: Core Platform Setup & Infrastructure
**Business Value**: Enables all subsequent development by establishing the fundamental project foundation, development environment, and core system architecture required for a secure, scalable microservice.
**Description**: This epic focuses on setting up the initial technical environment for the Authentication System, including establishing the microservice architecture, implementing basic infrastructure for horizontal scaling, redundancy, and ensuring secure communication protocols. It also covers the foundational monitoring and logging capabilities.
**UI Impact**: No
**Screen References**: N/A
**Key Deliverables**:
- Microservice boilerplate and project structure.
- CI/CD pipeline for automated builds and deployments.
- Development and staging environments.
- Basic API Gateway integration setup for RESTful API (TR-003).
- Implementation of HTTPS/TLS 1.2+ for all communication (TR-002).
- Initial monitoring and logging frameworks.
- Load balancing and horizontal scaling configuration.
**Dependent EPICs**: None (This is a foundational epic)
**Mapped Requirement IDs**: NFR-SCA-001, NFR-SCA-002, NFR-SCA-003, NFR-REL-001, NFR-REL-002, NFR-PER-002, NFR-PER-003, TR-002, TR-003, TR-004

### EP-DATA: User & Security Data Model
**Business Value**: Establishes the secure and persistent storage layer for all user and security-related data, which is critical for enabling any user-facing functionality.
**Description**: This epic defines and implements the database schema and persistence mechanisms for all core entities such as user accounts, verification tokens, password reset tokens, MFA configurations, login attempt logs, and session management. It ensures sensitive data, especially passwords, are stored using industry-standard secure hashing algorithms.
**UI Impact**: No
**Screen References**: N/A
**Key Deliverables**:
- Database schema design for User, VerificationToken, PasswordResetToken, MFASetting, LoginAttempt, Session/Token Blacklist entities.
- Implementation of data persistence layer (ORM/DAO).
- Secure password hashing implementation using bcrypt or Argon2 (NFR-SEC-002).
- Initial database migrations and seeding for mock data.
**Dependent EPICs**: EP-TECH
**Mapped Requirement IDs**: DR-001, DR-002, DR-003, DR-004, DR-005, DR-006, NFR-SEC-002

### EP-001: User Registration & Email Verification
**Business Value**: Enables new users to create accounts and activate them, forming the essential entry point into the system and expanding the user base.
**Description**: This epic covers the entire new user registration flow, including collecting user details, validating input (email format, password policy), ensuring unique email addresses, and the critical email verification process required for account activation.
**UI Impact**: Yes
**Screen References**: Registration Page, Email Verification Confirmation Page
**Key Deliverables**:
- User registration API endpoint.
- Email format validation logic.
- Password policy enforcement during registration.
- Unique email check mechanism.
- Email verification token generation and sending via external email service.
- Account activation logic via verification link.
- Error handling and user feedback for registration and verification.
**Dependent EPICs**: EP-TECH, EP-DATA
**Mapped Requirement IDs**: FR-REG-001, FR-REG-002, FR-REG-003, FR-REG-004, FR-REG-005, FR-REG-006, UC-REG-001, UC-REG-004

### EP-002: User Login & Account Security
**Business Value**: Provides the core functionality for users to access the system securely and implements immediate security measures like account lockout and brute-force protection to safeguard user accounts.
**Description**: This epic implements the primary login mechanism where users authenticate with their email and password. It also includes robust security features to counter brute-force attacks by introducing temporary account lockouts after multiple failed attempts and providing mechanisms for account unlocking.
**UI Impact**: Yes
**Screen References**: Login Page
**Key Deliverables**:
- User login API endpoint.
- Credential validation and authentication logic.
- Account status checks (active, pending verification, locked).
- Failed login attempt tracking.
- Account lockout mechanism based on consecutive failures (FR-LOG-002, FR-ALC-001).
- Account unlock mechanism (e.g., via password reset or automatic timeout) (FR-ALC-002).
- Brute-force protection, including rate limiting for login attempts (NFR-SEC-004).
- Error messages for invalid credentials and locked accounts.
**Dependent EPICs**: EP-TECH, EP-DATA, EP-001
**Mapped Requirement IDs**: FR-LOG-001, FR-LOG-002, FR-ALC-001, FR-ALC-002, UC-LOG-001, UC-ALC-001, NFR-PER-001, NFR-SEC-004

### EP-003: Password Management & Reset
**Business Value**: Enhances user experience and account security by providing robust self-service options for forgotten passwords and enforcing strong password policies for all password changes.
**Description**: This epic implements the full lifecycle of password management, including the ability for users to initiate and complete a password reset process when they forget their password. It also ensures that all new and updated passwords adhere to a predefined strong password policy.
**UI Impact**: Yes
**Screen References**: Forgot Password Page, Password Reset Form
**Key Deliverables**:
- Password reset initiation API endpoint.
- Password reset token generation and sending via email.
- Password reset completion API endpoint with token validation.
- Strong password policy enforcement for new passwords (FR-PWD-003).
- Invalidation of password reset tokens after use or expiration.
- User feedback for password reset process.
**Dependent EPICs**: EP-TECH, EP-DATA, EP-001, EP-002
**Mapped Requirement IDs**: FR-PWD-001, FR-PWD-002, FR-PWD-003, UC-PWD-001

### EP-004: Session Management & Logout
**Business Value**: Secures and manages active user sessions, ensuring authorized access duration, protection against session hijacking, and proper termination of user sessions.
**Description**: This epic focuses on implementing token-based authentication for managing user sessions, including issuing secure, time-limited tokens upon successful login. It also covers automatic session expiration, inactivity-based logouts, and explicit user-initiated logout functionality to invalidate sessions.
**UI Impact**: Yes (Logout button implies UI)
**Screen References**: N/A (Logout typically integrated into existing UI elements)
**Key Deliverables**:
- Implementation of token-based authentication (e.g., JWT).
- Generation and issuance of secure, signed, and time-limited authentication tokens.
- Token validation mechanism for integrated applications.
- Server-side session expiration logic.
- Inactivity detection and automated logout.
- Explicit logout API endpoint for token invalidation.
- Redirection to login page after logout/expiration.
**Dependent EPICs**: EP-TECH, EP-DATA, EP-002
**Mapped Requirement IDs**: FR-SES-001, FR-SES-002, FR-SES-003, FR-SES-004, UC-SES-004

### EP-005: Multi-Factor Authentication (MFA)
**Business Value**: Significantly enhances the security posture of user accounts by adding an extra layer of verification, reducing the risk of unauthorized access even if a password is compromised.
**Description**: This epic enables users to enroll in and utilize various Multi-Factor Authentication methods, such as Email OTP, SMS OTP, or Authenticator Apps. It covers the enrollment process, including verification of the selected method, and the subsequent MFA verification challenge during the login flow.
**UI Impact**: Yes
**Screen References**: MFA Enrollment Page, MFA Verification Prompt
**Key Deliverables**:
- MFA enrollment API endpoints for various methods (Email OTP, SMS OTP, Authenticator App).
- Logic for sending and verifying OTPs via email/SMS.
- QR code generation and secret key management for Authenticator Apps.
- Integration with external SMS service (Assumed A-002).
- MFA challenge integration into the login flow.
- Handling of invalid OTP attempts.
**Dependent EPICs**: EP-TECH, EP-DATA, EP-002
**Mapped Requirement IDs**: FR-MFA-001, FR-MFA-002, UC-MFA-001, UC-MFA-002

### EP-006: Overall System Security Compliance
**Business Value**: Ensures the entire Authentication System adheres to critical industry security standards, minimizing vulnerabilities and building trust with users and integrated applications.
**Description**: This epic focuses on ensuring that the system's design, implementation, and operational practices comply with the OWASP Top 10 guidelines and other defined security constraints. It includes activities like security audits, penetration testing preparation, and embedding secure coding practices across all development efforts.
**UI Impact**: No
**Screen References**: N/A
**Key Deliverables**:
- Security audit reports demonstrating OWASP Top 10 compliance (NFR-SEC-001).
- Secure coding guidelines and practices documentation.
- Integration of security scanning tools into the CI/CD pipeline.
- Mitigation strategies for identified vulnerabilities.
**Dependent EPICs**: All other feature epics (as compliance applies across the board)
**Mapped Requirement IDs**: NFR-SEC-001, TR-001
```