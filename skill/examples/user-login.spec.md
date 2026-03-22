---
feature: user-login
title: User Login Flow
created: 2026-03-22
updated: 2026-03-22
status: draft
depends_on: []
steps: 5
tags: [auth, core]
---

# E2E Spec: User Login Flow

Full round-trip: User enters credentials -> Server validates against PostgreSQL users table -> JWT returned -> User sees dashboard.

## e2e-login-001: Login Form Display

**Chain:** Screen
**Status:** pending

### What
User navigates to /login and sees a login form with email input, password input, and submit button.

### Verification Criteria
- [ ] Login form renders at /login route
- [ ] Email input exists with type="email" and placeholder
- [ ] Password input exists with type="password"
- [ ] Submit button exists and is initially enabled
- [ ] No error messages visible on initial load

### Details
- **Element:** Login form with 2 inputs (email, password) + 1 submit button
- **User Action:** Navigate to /login
- **Initial State:** Empty form, no error messages visible, submit button enabled

## e2e-login-002: Login Submission

**Chain:** Connection
**Status:** pending

### What
User fills email and password fields, then clicks submit. Frontend sends POST request to authentication endpoint with credentials.

### Verification Criteria
- [ ] Submit triggers POST /api/auth/login
- [ ] Request body contains {email: string, password: string}
- [ ] Authorization header is not required (login is pre-auth)
- [ ] Loading state shown while request is in-flight
- [ ] Submit button disabled during request

### Details
- **Method:** POST
- **Endpoint:** /api/auth/login
- **Request:** `{ email: string, password: string }`
- **Auth:** None (pre-authentication endpoint)

## e2e-login-003: Credential Validation

**Chain:** Processing
**Status:** pending

### What
Server receives login request, validates email exists in database, compares password hash using bcrypt, and generates JWT token on success.

### Verification Criteria
- [ ] Email lookup queries the users table
- [ ] Password comparison uses bcrypt (not plain text)
- [ ] JWT is generated with user ID payload
- [ ] JWT expiration is set (not infinite)
- [ ] Failed login does not reveal whether email or password was wrong

### Details
- **Steps:**
  1. Validate request body (email format, password non-empty)
  2. Query users table by email
  3. Compare submitted password with stored hash (bcrypt)
  4. Generate JWT with {userId, exp} payload
- **Storage:** PostgreSQL `users` table -- READ (SELECT by email)

## e2e-login-004: Successful Login Response

**Chain:** Response
**Status:** pending

### What
Server returns JWT token and user profile. Frontend stores token and redirects user to dashboard with personalized greeting.

### Verification Criteria
- [ ] Response status is 200
- [ ] Response body contains {token: string, user: {id, email, name}}
- [ ] Token is stored in localStorage or httpOnly cookie
- [ ] Browser redirects to /dashboard
- [ ] Dashboard shows user's name in welcome message

### Details
- **Success Status:** 200 OK
- **Response Shape:** `{ token: string, user: { id: string, email: string, name: string } }`
- **UI Updates:**
  - JWT token stored in client (localStorage or httpOnly cookie)
  - Browser navigates to /dashboard
  - Dashboard header shows "Welcome, {name}"
  - Login form is no longer accessible (redirects to /dashboard if already authenticated)

## e2e-login-005: Login Error Handling

**Chain:** Error
**Status:** pending

### What
All failure modes for the login flow, with specific user-visible behavior for each condition.

### Verification Criteria
- [ ] Empty email/password shows inline validation error (no API call made)
- [ ] Invalid credentials show "Invalid email or password" error message
- [ ] Server error shows "Something went wrong. Please try again." toast
- [ ] Network timeout shows "Connection failed. Check your internet." message
- [ ] Rate limit (429) shows "Too many attempts. Try again in X minutes."

### Details
| Condition | Behavior | Status |
|-----------|----------|--------|
| Empty email or password | Inline validation: "Email is required" / "Password is required" | null (client-side) |
| Invalid email format | Inline validation: "Please enter a valid email address" | null (client-side) |
| Invalid credentials | Error message below form: "Invalid email or password". Clear password field. | 401 |
| Server error | Toast notification: "Something went wrong. Please try again." | 500 |
| Network timeout | Error message: "Connection failed. Check your internet." | null (network) |
| Rate limited | Error message: "Too many attempts. Try again in {minutes} minutes." | 429 |

## Deviations

_No deviations recorded yet._
