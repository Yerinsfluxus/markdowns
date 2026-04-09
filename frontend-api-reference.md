# Riverly API — Frontend Reference
> Base URL: `https://riverly-api-staging.duckdns.org`
> Interactive docs: `https://riverly-api-staging.duckdns.org/swagger`
> Last updated: April 9, 2026

---

## Global Conventions

### Every response has this wrapper:
```json
{
  "status": true,
  "message": "Request completed successfully.",
  "data": { }
}
```
- `status: true` = success, `status: false` = failure
- `data` is always inside the wrapper, not at the root level

### Auth header (for all protected endpoints):
```
Authorization: Bearer <accessToken>
```

### Rate limits:
| Policy | Applies to | Limit |
|--------|-----------|-------|
| `auth` | Login, new-device login | 10 req/min |
| `otp` | Initiate phone OTP | 5 req/min |
| `ip_policy` | All endpoints | 20 req/min per IP |

Returns **HTTP 429** when exceeded.

### Device Headers (required where marked ✅):
```
X-Device-Id: <unique UUID, generated once on install>   ← REQUIRED
X-Device-Name: "iPhone 14 Pro"                          ← optional
X-Device-Type: "ios" | "android" | "web"                ← optional
X-Device-Os: "iOS 17.1"                                 ← optional
```

---

## 1. Auth — `/api/v1/auth`

### POST `/api/v1/auth/login`
Login with passcode. Device headers required.

**Request:**
```json
{
  "identifier": "+2348012345678",   // phone or email — field is "identifier" NOT "phoneNumber"
  "passcode": "123456"
}
```

**Response `data`:**
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "...",
  "userId": "guid",
  "name": "Fatai Sanni",
  "phoneNumber": "+2348012345678",
  "onboardingStep": "completed",
  "deviceVerificationRequired": false,
  "deviceVerificationMessage": null
}
```
> If `deviceVerificationRequired: true`, redirect to verify-device flow before proceeding.

---

### POST `/api/v1/auth/login/new-device`
Login attempt from an unrecognised device — triggers OTP to phone. Device headers required.

**Request:**
```json
{
  "identifier": "+2348012345678"   // phone only
}
```

**Response `data`:**
```json
{
  "userId": "guid",
  "deviceVerificationRequired": true,
  "deviceVerificationMessage": "OTP sent to your phone"
}
```

---

### POST `/api/v1/auth/login/verify-device`
Verify the OTP sent during new-device login. Device headers required.

**Request:**
```json
{
  "userId": "guid",
  "otpCode": "123456"
}
```

---

### POST `/api/v1/auth/refresh`
Get a new access token using a refresh token.

**Request:**
```json
{
  "refreshToken": "..."
}
```

**Response `data`:**
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "..."
}
```

---

### POST `/api/v1/auth/logout`
🔒 **Requires auth.** Revokes refresh token.

**Request:**
```json
{
  "refreshToken": "..."
}
```

---

### POST `/api/v1/auth/passcode/forgot`
Initiate passcode reset — sends OTP to phone/email.

**Request:**
```json
{
  "identifier": "+2348012345678"   // phone or email
}
```

---

### POST `/api/v1/auth/passcode/verify-otp`
Verify OTP from forgot-passcode flow — returns a reset token.

**Request:**
```json
{
  "identifier": "+2348012345678",
  "otpCode": "123456"
}
```

**Response `data`:**
```json
{
  "resetToken": "..."
}
```

---

### POST `/api/v1/auth/passcode/reset`
Reset passcode using the token from verify-otp.

**Request:**
```json
{
  "identifier": "+2348012345678",
  "resetToken": "...",
  "newPasscode": "654321",         // exactly 6 digits
  "confirmNewPasscode": "654321"
}
```

---

### POST `/api/v1/auth/passcode/change`
🔒 **Requires auth.** Change passcode when already logged in.

**Request:**
```json
{
  "currentPasscode": "123456",
  "newPasscode": "654321",         // exactly 6 digits
  "confirmNewPasscode": "654321"
}
```

---

### GET `/api/v1/auth/user/profile`
Get user profile. No auth required on the route but needs a valid JWT to return user data.

**Response `data`:**
```json
{
  "userId": "guid",
  "firstName": "Fatai",
  "lastName": "Sanni",
  "email": "fatai@example.com",
  "phoneNumber": "+2348012345678",
  "isEmailVerified": true,
  "isPhoneVerified": true,
  "state": "Lagos",
  "lga": "Ikeja",
  "address": "...",
  "profession": "...",
  "incomeRange": "...",
  "kycStatus": "...",
  "status": "...",
  "lastLoginAt": "2026-04-09T...",
  "createdAt": "2026-04-01T...",
  "accountNumber": "...",
  "accountName": "...",
  "bankName": "...",
  "currency": "NGN",
  "tier": "...",
  "dailyTransactionLimit": 100000,
  "singleTransactionLimit": 50000
}
```

---

### POST `/api/v1/auth/biometric/enable`
🔒 **Requires auth.** Register device for biometric login.

**Request:**
```json
{
  "deviceId": "uuid-of-device"
}
```

**Response `data`:**
```json
{
  "deviceId": "uuid",
  "biometricToken": "..."    // store this token on-device for biometric login
}
```

---

### POST `/api/v1/auth/biometric/login`
Login using stored biometric token. Device headers required.

**Request:**
```json
{
  "deviceId": "guid",
  "biometricToken": "..."
}
```

---

### GET `/api/v1/auth/biometric/status`
🔒 **Requires auth.** Check if biometric is enabled for the current user.

### POST `/api/v1/auth/biometric/disable`
🔒 **Requires auth.** Revoke biometric for current user.

### POST `/api/v1/auth/device/revoke`
🔒 **Requires auth.** Revoke a specific device session.

**Request:**
```json
{
  "deviceId": "guid"
}
```

---

## 2. Onboarding — `/api/v1/onboarding`

> Step order: Phone → Verify Phone → Email → Verify Email → Identity → Profile → Passcode → Liveness

### POST `/api/v1/onboarding/phone/initiate`
Step 1 — Start signup with phone number. Sends OTP.

**Request:**
```json
{
  "phoneNumber": "+2348012345678",
  "referralCode": "ABC123"          // optional
}
```

---

### POST `/api/v1/onboarding/phone/verify`
Step 2 — Verify phone OTP. Device headers required. Returns a JWT for subsequent onboarding steps.

**Request:**
```json
{
  "target": "+2348012345678",       // field is "target", NOT "phoneNumber"
  "code": "123456"                  // field is "code", NOT "otp"
}
```

**Response `data`:**
```json
{
  "accessToken": "eyJ...",
  "userId": "guid",
  "onboardingStep": "email"
}
```

---

### POST `/api/v1/onboarding/email/submit`
Step 3 — Submit email address. 🔒 Requires onboarding JWT.

**Request:**
```json
{
  "email": "user@example.com"
}
```

---

### POST `/api/v1/onboarding/email/verify`
Step 4 — Verify email OTP. 🔒 Requires auth. Device headers required.

**Request:**
```json
{
  "target": "user@example.com",     // field is "target"
  "code": "123456"                  // field is "code"
}
```

---

### POST `/api/v1/onboarding/resend-otp`
Resend OTP for phone or email.

**Request:**
```json
{
  "identifier": "+2348012345678"    // phone or email
}
```

---

### POST `/api/v1/onboarding/identity/verify`
Step 5 — BVN or NIN verification. 🔒 Requires auth.

**Request:**
```json
{
  "bvn": "12345678901",            // provide one or both
  "nin": "12345678901"
}
```

---

### POST `/api/v1/onboarding/profile/update`
Step 6 — Submit residential and professional details. 🔒 Requires auth.

**Request:**
```json
{
  "state": "Lagos",
  "lga": "Ikeja",
  "area": "GRA",
  "address": "14 Street Name",
  "zipCode": "100001",
  "profession": "Software Engineer",
  "annualIncomeRange": "1000000-5000000"   // field is "annualIncomeRange"
}
```

---

### POST `/api/v1/onboarding/passcode/set`
Step 7 — Set login passcode. 🔒 Requires auth.

**Request:**
```json
{
  "passcode": "123456",            // exactly 6 digits
  "confirmPasscode": "123456"
}
```

---

### POST `/api/v1/onboarding/liveness/check`
Step 8 — Facial liveness check. 🔒 Requires auth.

**Request:**
```json
{
  "image": "base64encodedstring..."    // base64-encoded image from camera
}
```

---

## 3. Accounts — `/api/v1/accounts`
🔒 All endpoints require auth.

### POST `/api/v1/accounts`
Create a bank account for the user.

**Request:**
```json
{
  "accountProductCode": "12",
  "branchCode": "101",
  "accountOfficerCode": "10",
  "currency": "NGN",
  "oldAccountNo": null             // optional
}
```

---

### GET `/api/v1/accounts`
Get all user accounts with balances.

**Response `data`:** Array of account objects with balances.

---

### GET `/api/v1/accounts/{accountId}/balance`
Get balance for a single account.

---

### POST `/api/v1/accounts/set-pin`
Set a 4-digit transaction PIN.

**Request:**
```json
{
  "pin": "1234",                   // exactly 4 digits
  "confirmPin": "1234"
}
```

---

### POST `/api/v1/accounts/change-pin`
Change transaction PIN.

**Request:**
```json
{
  "currentPin": "1234",
  "newPin": "5678",
  "confirmNewPin": "5678"
}
```

---

### POST `/api/v1/accounts/forgot-pin`
Reset PIN using passcode.

**Request:**
```json
{
  "passcode": "123456",
  "newPin": "5678",
  "confirmNewPin": "5678"
}
```

---

### POST `/api/v1/accounts/{accountId}/verify`
Verify transaction PIN before a transaction.

**Request:**
```json
{
  "pin": "1234"
}
```

---

## 4. Transactions — `/api/v1/accounts/{accountId}/transactions`
🔒 All endpoints require auth.

### GET `/api/v1/accounts/{accountId}/transactions`
Get paginated transaction history. All filters are query params.

**Query params (all optional):**
```
?page=1
&pageSize=20
&dateFrom=2026-01-01
&dateTo=2026-04-01
&type=debit                        // TransactionType enum
&status=successful
&direction=outbound
&category=transfer
&amountMin=1000
&amountMax=50000
&reference=TXN12345
&search=john                       // searches narration + counterparty name
```

**Response `data`:**
```json
{
  "items": [],
  "page": 1,
  "pageSize": 20,
  "totalCount": 0,
  "totalPages": 0,
  "hasNextPage": false,
  "hasPreviousPage": false
}
```

---

### GET `/api/v1/accounts/{accountId}/transactions/{transactionId}`
Get single transaction detail / receipt.

### GET `/api/v1/transactions/{reference}`
Look up any transaction by reference string.

---

## 5. Transfers — `/api/v1/transfers`
🔒 All endpoints require auth.

### GET `/api/v1/transfers/internal-lookup/{accountNumber}`
Look up a Riverly account before transferring.

**Response `data`:** Account name and number if found.

---

### POST `/api/v1/transfers/internal`
Transfer funds between Riverly accounts.

**Request:**
```json
{
  "sourceAccountId": "guid",
  "destinationAccountNumber": "0123456789",
  "amount": 5000.00,
  "narration": "Rent payment",         // optional, max 100 chars
  "transactionPin": "1234",
  "idempotencyKey": "uuid-v4"          // generate a new UUID per transfer attempt
}
```

---

### GET `/api/v1/transfers/receipt/{reference}`
Get transfer receipt by reference.

---

## 6. Beneficiaries — `/api/v1/beneficiaries`
🔒 All endpoints require auth.

### GET `/api/v1/beneficiaries/banks`
Get list of all supported banks (name + bank code).

### POST `/api/v1/beneficiaries/validate`
Validate an external bank account before saving/transferring.

**Request:**
```json
{
  "accountNumber": "0123456789",
  "bankCode": "058"                // from /banks list
}
```

### GET `/api/v1/beneficiaries`
List all saved beneficiaries.

**Query params (all optional):**
```
?search=john
&type=internal                     // BeneficiaryType enum: internal | external
&isFavorite=true
&page=1
&pageSize=20
```

### GET `/api/v1/beneficiaries/{beneficiaryId}`
Get single beneficiary.

### POST `/api/v1/beneficiaries`
Save a new beneficiary.

**Request:**
```json
{
  "accountNumber": "0123456789",
  "bankCode": "058",
  "nickname": "Ahmed GTB",         // optional
  "isFavorite": false
}
```

### PATCH `/api/v1/beneficiaries/{beneficiaryId}`
Update nickname or favourite status.

**Request:**
```json
{
  "nickname": "Ahmed",             // optional
  "isFavorite": true               // optional
}
```

### DELETE `/api/v1/beneficiaries/{beneficiaryId}`
Remove a beneficiary (soft delete).

---

## 7. Bill Payments — `/api/v1/payments`
🔒 All endpoints require auth.

### GET `/api/v1/payments/data/plans/{networkProvider}`
Get data plans for a network. `networkProvider` = `mtn` | `glo` | `airtel` | `9mobile`

### GET `/api/v1/payments/bills/categories`
Get all biller categories (electricity, cable TV, etc.).

### GET `/api/v1/payments/bills/{category}`
Get all billers within a category.

### POST `/api/v1/payments/bills/validate`
Validate a customer reference (meter number, smartcard, etc.) before payment.

**Request:**
```json
{
  "billerId": "IKEDC",
  "customerReference": "45062000001"
}
```

### POST `/api/v1/payments/airtime`
Purchase airtime.

**Request:**
```json
{
  "accountId": "guid",
  "phoneNumber": "+2348012345678",
  "networkProvider": "mtn",         // mtn | glo | airtel | 9mobile
  "amount": 1000,                   // min 50, max 500000
  "transactionPin": "1234",
  "idempotencyKey": "uuid-v4"
}
```

### POST `/api/v1/payments/data`
Purchase data bundle.

**Request:**
```json
{
  "accountId": "guid",
  "phoneNumber": "+2348012345678",
  "networkProvider": "mtn",
  "planId": "provider-specific-id",  // from data plans endpoint
  "amount": 2000,
  "transactionPin": "1234",
  "idempotencyKey": "uuid-v4"
}
```

### POST `/api/v1/payments/bills/pay`
Pay a bill.

**Request:**
```json
{
  "accountId": "guid",
  "billerId": "IKEDC",
  "customerReference": "45062000001",
  "amount": 5000,
  "transactionPin": "1234",
  "idempotencyKey": "uuid-v4",
  "billerPlanId": null              // optional, for cable TV bouquets
}
```

### GET `/api/v1/payments/airtime/{reference}/receipt`
### GET `/api/v1/payments/data/{reference}/receipt`
### GET `/api/v1/payments/bills/{reference}/receipt`
Get receipts by transaction reference.

---

## 8. KYC — `/api/v1/kyc`
🔒 All endpoints require auth.

### POST `/api/v1/kyc/documents/upload`
Upload a KYC document. **Must be `multipart/form-data`**, max 5 MB.

**Form fields:**
```
File           → the document file
DocumentType   → enum value (NIN slip, passport, driver's license, etc.)
Side           → enum value (front/back)
```

### POST `/api/v1/kyc/submit`
Submit KYC for review after documents are uploaded.

### GET `/api/v1/kyc/status`
Get current KYC verification status and tier.

---

## 9. Dashboard — `/api/v1/dashboard`
🔒 Requires auth.

### GET `/api/v1/dashboard`
Main dashboard data — accounts, balances, recent transactions, KYC status. Response is cached for **30 seconds per user**.

---

## 10. Statements — `/api/v1/statements`
🔒 All endpoints require auth.

### POST `/api/v1/statements/request`
Request a bank statement (async — check status after).

**Request:**
```json
{
  "accountId": "guid",
  "dateFrom": "2026-01-01T00:00:00Z",
  "dateTo": "2026-04-01T00:00:00Z",
  "delivery": "Download"            // StatementDelivery enum: Download | Email
}
```

### GET `/api/v1/statements/{requestId}/status`
Check statement status and get download URL when ready.

---

## 11. Notifications — `/api/v1/notifications`
🔒 All endpoints require auth.

### POST `/api/v1/notifications/push-token`
Register device push token (call after login on mobile).

**Request:**
```json
{
  "deviceId": "uuid",
  "token": "firebase-or-apns-token",
  "platform": "ios"                 // ios | android | web
}
```

### DELETE `/api/v1/notifications/push-token/{deviceId}`
Deregister push token on logout.

### GET `/api/v1/notifications/preferences`
Get notification preferences.

### PATCH `/api/v1/notifications/preferences`
Update notification preferences (all fields optional — partial update).

**Request:**
```json
{
  "pushEnabled": true,
  "emailEnabled": true,
  "smsEnabled": false,
  "transferAlerts": true,
  "billPaymentAlerts": true,
  "cardAlerts": true,
  "loanAlerts": false,
  "securityAlerts": true,
  "promotionalAlerts": false
}
```

---

## 12. Chat — `/api/v1/chat`
🔒 All endpoints require auth.

### POST `/api/v1/chat/sessions`
Start a new support chat session. Returns **HTTP 201**.

**Request:**
```json
{
  "subject": "Issue with transfer"   // optional
}
```

### POST `/api/v1/chat/messages`
Send a message in an existing session.

**Request:**
```json
{
  "sessionId": "guid",
  "content": "Hello, I need help..."   // max 2000 chars
}
```

### GET `/api/v1/chat/sessions`
Get all chat sessions for the user.

### GET `/api/v1/chat/sessions/{sessionId}/messages`
Get all messages in a session.

### POST `/api/v1/chat/sessions/{sessionId}/close`
Close a chat session.

---

## 13. Support — `/api/v1/support`
🔒 Requires auth.

### GET `/api/v1/support/contact`
Get customer support contact options (phone, email, etc.).

---

## Quick Reference — Field Names to Watch

| Common mistake | Correct field name |
|---------------|-------------------|
| `phoneNumber` on login | `identifier` |
| `otp` / `otpCode` | `code` (on /onboarding/phone/verify and /email/verify) |
| `phoneNumber` on verify | `target` |
| `token` on login response | `accessToken` |
| `pin` as 6 digits | PIN is `4 digits`, passcode is `6 digits` |
| `amount` as string | `amount` must be a **number** (decimal) |
| Missing `idempotencyKey` | Required on transfers, airtime, data, and bill payments |
| Missing `X-Device-Id` header | Required on login, new-device login, phone verify, email verify, biometric login |
