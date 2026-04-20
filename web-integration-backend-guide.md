# Riverly Web (Corporate Internet Banking) — Backend Integration Guide

> **Status:** Architecture & Planning  
> **Audience:** Backend engineers working on the Riverly .NET API  
> **Last Updated:** April 2026

---

## 1. Overview

Riverly's Corporate Internet Banking is a React/Next.js web platform that provides maker-checker workflows, bulk transfers, RBAC, reporting, and audit trails for corporate/business users.

**The answer to "do we need a separate backend?" is NO.**

The existing `.NET API` (`Riverly.Api`) serves both mobile and web. No separate project, no separate deployment. The web platform consumes the same API — the only additions are:

1. A new **route prefix** for corporate-only endpoints
2. New **domain entities, services, and controllers** for corporate features
3. **RBAC policies** on top of the existing JWT authentication
4. A **corporate-scoped JWT claim** so every request carries organization context

---

## 2. Channel Strategy

| Channel | Client | Route Prefix | Auth |
|---|---|---|---|
| Mobile Banking (Retail) | Flutter (iOS/Android) | `api/v1/` | JWT — retail user |
| Corporate Internet Banking | React/Next.js | `api/v1/corporate/` | JWT — corporate user with `OrganizationId` + `Role` claims |

Shared endpoints (auth, transfers, bills, accounts) remain at `api/v1/` — corporate web users call these too. The `api/v1/corporate/` prefix is exclusively for features that only exist in the corporate context.

CORS is already partially configured in `Program.cs`:
```csharp
policy.WithOrigins(
    "https://app.riverly.com",
    "https://admin.riverly.com",  // ← web platform already here
    "http://localhost:3000",
    "http://localhost:4200")
```
No CORS change needed.

---

## 3. Corporate Endpoint Map

All new corporate endpoints follow this pattern: `api/v1/corporate/{resource}`

### 3.1 Authentication (Corporate)
```
POST   api/v1/corporate/auth/login
POST   api/v1/corporate/auth/refresh
POST   api/v1/corporate/auth/logout
POST   api/v1/corporate/auth/otp/verify
```
Corporate login is a separate flow from retail login. It issues a JWT that includes `OrganizationId`, `CorporateUserId`, and `Role` claims. The existing `api/v1/auth` retail login endpoints are untouched.

### 3.2 Organization & User Management
```
GET    api/v1/corporate/users                    ← list org users (Admin only)
POST   api/v1/corporate/users                    ← invite/create user (Admin only)
GET    api/v1/corporate/users/{userId}
PUT    api/v1/corporate/users/{userId}
DELETE api/v1/corporate/users/{userId}           ← deactivate (Admin only)

GET    api/v1/corporate/roles                    ← list roles
POST   api/v1/corporate/roles                    ← create role (Admin only)
PUT    api/v1/corporate/roles/{roleId}
DELETE api/v1/corporate/roles/{roleId}

PUT    api/v1/corporate/users/{userId}/role      ← assign role to user (Admin only)
```

### 3.3 Maker-Checker (Approvals)
```
GET    api/v1/corporate/approvals                ← pending queue (Checker/Admin)
GET    api/v1/corporate/approvals/{id}           ← detail view
POST   api/v1/corporate/approvals/{id}/approve   ← Checker approves
POST   api/v1/corporate/approvals/{id}/reject    ← Checker rejects (with comment)
POST   api/v1/corporate/approvals/{id}/return    ← Checker returns to Maker for correction
```

### 3.4 Corporate Transfers (Maker submits here)
```
POST   api/v1/corporate/transfers/single         ← single transfer, enters approval queue
POST   api/v1/corporate/transfers/bulk           ← bulk upload, enters approval queue
GET    api/v1/corporate/transfers                ← transfer history for the org
GET    api/v1/corporate/transfers/{id}
GET    api/v1/corporate/transfers/{id}/receipt
```
Note: These do NOT execute immediately. They create an `ApprovalRequest` record with status `PendingApproval`. Execution only happens after a Checker approves via the approvals endpoints above.

### 3.5 Bulk Transfers
```
POST   api/v1/corporate/bulk-transfers/validate  ← validate file before submission
POST   api/v1/corporate/bulk-transfers           ← submit validated batch (enters queue)
GET    api/v1/corporate/bulk-transfers/{id}      ← batch status and line-item results
GET    api/v1/corporate/bulk-transfers/{id}/errors ← failed rows only
```

### 3.6 Reports & Audit Logs
```
GET    api/v1/corporate/reports/transactions     ← filterable transaction report
GET    api/v1/corporate/reports/balances         ← account balances snapshot
GET    api/v1/corporate/reports/audit-logs       ← user activity and approval history
GET    api/v1/corporate/reports/payroll-summary  ← payroll batch summary
POST   api/v1/corporate/reports/export           ← trigger CSV/PDF/Excel export
GET    api/v1/corporate/reports/export/{jobId}   ← poll or download export result
```

### 3.7 Dashboard (Corporate)
```
GET    api/v1/corporate/dashboard                ← org balances, pending approvals count, recent activity
```

---

## 4. Shared Endpoints (Used by Both Mobile and Web)

These endpoints at `api/v1/` are consumed by both channels without any change. Corporate web users call them the same way retail mobile users do:

| Endpoint | Used for |
|---|---|
| `api/v1/transfers/name-enquiry` | Recipient name lookup before transfer confirmation |
| `api/v1/bills/*` | Bill payments (if corporate uses bill pay) |
| `api/v1/accounts/*` | Account balance and details |
| `api/v1/transactions/*` | Transaction history |
| `api/v1/beneficiaries/*` | Saved beneficiaries |
| `api/v1/notifications/*` | Notification preferences |

---

## 5. JWT Token Structure for Corporate Users

The corporate JWT must carry additional claims compared to the retail token. The `IJwtService` will need a new method (e.g., `GenerateCorporateToken`) or an overload.

**Retail JWT claims (current):**
```json
{
  "sub": "user-guid",
  "name": "Tobi Adeyemi",
  "role": "RetailUser"
}
```

**Corporate JWT claims (new):**
```json
{
  "sub": "corporate-user-guid",
  "name": "Kunle Balogun",
  "org_id": "organization-guid",
  "org_name": "Acme Ltd",
  "role": "Checker",
  "channel": "web"
}
```

The `org_id` claim is critical — every corporate service method receives it to scope queries to the correct organization. A Checker at Org A must never be able to see or act on Org B's approval queue.

**How to read it in a controller (via `BaseController`):**
```csharp
protected Guid GetOrganizationId() =>
    Guid.Parse(User.FindFirstValue("org_id")!);

protected string GetCorporateRole() =>
    User.FindFirstValue(ClaimTypes.Role)!;
```
Add these two helpers to `BaseController.cs` when implementing.

---

## 6. RBAC — Role-Based Access Control

### 6.1 Corporate Roles (from PRD Section 8.1)
| Role | Permissions |
|---|---|
| `Admin` | Manage users, roles, limits, settings, beneficiaries, reports. Full read/write. |
| `Maker` | Initiate transfers (single/bulk), submit for approval. Cannot approve own transactions. |
| `Checker` | Approve, reject, or return transactions. Cannot initiate. |
| `Viewer` | Read-only across all org data. |

### 6.2 Policy Setup in `Program.cs`
When implementing, register these authorization policies alongside the existing JWT auth setup:

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CorporateAdmin",   p => p.RequireRole("Admin"));
    options.AddPolicy("MakerOnly",        p => p.RequireRole("Maker"));
    options.AddPolicy("CheckerOnly",      p => p.RequireRole("Checker"));
    options.AddPolicy("CorporateViewer",  p => p.RequireRole("Admin", "Maker", "Checker", "Viewer"));
    options.AddPolicy("CorporateAccess",  p => p.RequireRole("Admin", "Maker", "Checker", "Viewer"));
});
```

### 6.3 Applying Policies on Controllers
```csharp
[HttpPost("approve")]
[Authorize(Policy = "CheckerOnly")]
public async Task<IActionResult> Approve(Guid id, [FromBody] ApprovalActionRequest request) { ... }

[HttpPost("transfers/single")]
[Authorize(Policy = "MakerOnly")]
public async Task<IActionResult> SubmitTransfer([FromBody] CorporateTransferRequest request) { ... }

[HttpGet("users")]
[Authorize(Policy = "CorporateAdmin")]
public async Task<IActionResult> GetUsers() { ... }
```

---

## 7. New Domain Entities Required

These need to be created in `Riverly.Domain/Entities/` and migrated via EF Core.

### `Organization`
```csharp
// Riverly.Domain/Entities/Organization.cs
public class Organization
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string RegistrationNumber { get; set; }
    public string Email { get; set; }
    public OrganizationStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public ICollection<CorporateUser> Users { get; set; }
    public ICollection<ApprovalRequest> ApprovalRequests { get; set; }
}
```

### `CorporateUser`
```csharp
// Riverly.Domain/Entities/CorporateUser.cs
public class CorporateUser
{
    public Guid Id { get; set; }
    public Guid OrganizationId { get; set; }
    public Organization Organization { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public string PasswordHash { get; set; }
    public CorporateRole Role { get; set; }       // Admin | Maker | Checker | Viewer
    public bool IsActive { get; set; }
    public DateTime? LastLoginAt { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

### `ApprovalRequest`
```csharp
// Riverly.Domain/Entities/ApprovalRequest.cs
public class ApprovalRequest
{
    public Guid Id { get; set; }
    public Guid OrganizationId { get; set; }
    public Guid InitiatedBy { get; set; }           // Maker's CorporateUserId
    public Guid? ReviewedBy { get; set; }           // Checker's CorporateUserId
    public ApprovalRequestType Type { get; set; }   // SingleTransfer | BulkTransfer
    public ApprovalStatus Status { get; set; }      // PendingApproval | Approved | Rejected | Returned
    public string PayloadJson { get; set; }         // serialized transfer details
    public string? ReviewComment { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ReviewedAt { get; set; }
    public ICollection<AuditEntry> AuditTrail { get; set; }
}
```

### `AuditEntry`
```csharp
// Riverly.Domain/Entities/AuditEntry.cs
public class AuditEntry
{
    public Guid Id { get; set; }
    public Guid OrganizationId { get; set; }
    public Guid ActorId { get; set; }
    public string ActorName { get; set; }
    public string Action { get; set; }              // e.g. "ApprovalRequest.Approved"
    public string EntityType { get; set; }
    public Guid? EntityId { get; set; }
    public string? Notes { get; set; }
    public string IpAddress { get; set; }
    public DateTime Timestamp { get; set; }
}
```

### `BulkTransferBatch`
```csharp
// Riverly.Domain/Entities/BulkTransferBatch.cs
public class BulkTransferBatch
{
    public Guid Id { get; set; }
    public Guid OrganizationId { get; set; }
    public Guid ApprovalRequestId { get; set; }
    public int TotalRows { get; set; }
    public int SuccessCount { get; set; }
    public int FailureCount { get; set; }
    public BatchStatus Status { get; set; }
    public ICollection<BulkTransferLine> Lines { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

---

## 8. New Services Required

Create interfaces in `Riverly.Application/Interfaces/` and implementations in `Riverly.Infrastructure/Services/`.

| Interface | Purpose |
|---|---|
| `ICorporateAuthService` | Corporate login, token issuance, session management |
| `ICorporateUserService` | CRUD for corporate users, role assignment |
| `IApprovalService` | Maker-checker workflow: submit, approve, reject, return |
| `IBulkTransferService` | Bulk file validation, batch creation and execution post-approval |
| `IAuditService` | Write and query audit log entries |
| `ICorporateReportService` | Generate and export transaction/balance/audit reports |
| `ICorporateDashboardService` | Org-level summary: pending approvals count, recent transfers, balances |

---

## 9. New Controllers to Create

All in `Riverly.Api/Controllers/Corporate/` (a subfolder for organization — no functional difference):

| Controller | Route |
|---|---|
| `CorporateAuthController` | `api/v1/corporate/auth` |
| `CorporateUsersController` | `api/v1/corporate/users` |
| `CorporateRolesController` | `api/v1/corporate/roles` |
| `ApprovalsController` | `api/v1/corporate/approvals` |
| `CorporateTransferController` | `api/v1/corporate/transfers` |
| `BulkTransferController` | `api/v1/corporate/bulk-transfers` |
| `CorporateReportsController` | `api/v1/corporate/reports` |
| `CorporateDashboardController` | `api/v1/corporate/dashboard` |

All controllers inherit `BaseController` exactly as the existing ones do.

**Controller template to follow:**
```csharp
// src/Riverly.Api/Controllers/Corporate/ApprovalsController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Riverly.Api.Common;
using Riverly.Application.DTOs.Corporate.Approvals;
using Riverly.Application.Interfaces;

namespace Riverly.Api.Controllers.Corporate;

[ApiController]
[Route("api/v1/corporate/approvals")]
[Authorize]
public class ApprovalsController : BaseController
{
    private readonly IApprovalService _approvalService;

    public ApprovalsController(IApprovalService approvalService)
    {
        _approvalService = approvalService;
    }

    [HttpGet]
    [Authorize(Policy = "CheckerOnly")]
    public async Task<IActionResult> GetPendingQueue()
    {
        var orgId = GetOrganizationId();
        var result = await _approvalService.GetPendingAsync(orgId);
        return Ok(result);
    }

    [HttpPost("{id:guid}/approve")]
    [Authorize(Policy = "CheckerOnly")]
    public async Task<IActionResult> Approve(Guid id, [FromBody] ApprovalActionRequest request)
    {
        var orgId = GetOrganizationId();
        var checkerId = GetUserId();
        var result = await _approvalService.ApproveAsync(id, orgId, checkerId, request);
        return Ok(result);
    }

    [HttpPost("{id:guid}/reject")]
    [Authorize(Policy = "CheckerOnly")]
    public async Task<IActionResult> Reject(Guid id, [FromBody] ApprovalActionRequest request)
    {
        var orgId = GetOrganizationId();
        var checkerId = GetUserId();
        var result = await _approvalService.RejectAsync(id, orgId, checkerId, request);
        return Ok(result);
    }
}
```

---

## 10. Maker-Checker Workflow — How It Works End to End

```
Maker submits transfer
        │
        ▼
POST api/v1/corporate/transfers/single
        │
        ▼
ApprovalRequest created (Status = PendingApproval)
Notification sent to all Checkers in the org
        │
        ▼
Checker opens GET api/v1/corporate/approvals
        │
        ▼
Checker reviews detail at GET api/v1/corporate/approvals/{id}
        │
   ┌────┴────────┬───────────────┐
   ▼             ▼               ▼
Approve       Reject           Return
   │             │               │
   ▼             ▼               ▼
Execute      Status =         Status =
transfer     Rejected        Returned
via Core     Notify          Notify
Banking/     Maker           Maker
NIBSS
   │
   ▼
Receipt generated
AuditEntry written
Maker notified of success/failure
```

**Key rule enforced in `IApprovalService`:** A Maker cannot approve their own submission. The service checks `request.InitiatedBy != checkerId` and returns a failure result if they match.

---

## 11. DTOs Folder Structure

New DTOs go in `Riverly.Application/DTOs/Corporate/`:

```
DTOs/
  Corporate/
    Auth/
      CorporateLoginRequest.cs
      CorporateLoginResponse.cs
    Users/
      CreateCorporateUserRequest.cs
      CorporateUserResponse.cs
      AssignRoleRequest.cs
    Approvals/
      ApprovalActionRequest.cs        ← contains Comment field
      ApprovalRequestResponse.cs
      ApprovalQueueResponse.cs
    Transfers/
      CorporateTransferRequest.cs
      BulkTransferSubmitRequest.cs
      BulkTransferValidationResult.cs
    Reports/
      ReportFilterRequest.cs
      ExportRequest.cs
      ExportJobResponse.cs
```

---

## 12. DI Registration Pattern

When adding the new services in `Program.cs`, follow the exact same pattern as the existing services:

```csharp
// Corporate Services — add after existing service registrations
builder.Services.AddScoped<ICorporateAuthService, CorporateAuthService>();
builder.Services.AddScoped<ICorporateUserService, CorporateUserService>();
builder.Services.AddScoped<IApprovalService, ApprovalService>();
builder.Services.AddScoped<IBulkTransferService, BulkTransferService>();
builder.Services.AddScoped<IAuditService, AuditService>();
builder.Services.AddScoped<ICorporateReportService, CorporateReportService>();
builder.Services.AddScoped<ICorporateDashboardService, CorporateDashboardService>();
```

---

## 13. Database Migrations

All new entities go through EF Core migrations in `Riverly.Infrastructure/Migrations/` as usual. When adding:
1. Add entity class in `Riverly.Domain/Entities/`
2. Add `DbSet<T>` in `ApplicationDbContext`
3. Run `dotnet ef migrations add <MigrationName> --project Riverly.Infrastructure --startup-project Riverly.Api`

---

## 14. What Is NOT Changing

To be explicit — none of the following change when web integration is added:

- All existing `api/v1/` retail endpoints remain untouched
- `BaseController.cs` — only adding two new helper methods (`GetOrganizationId`, `GetCorporateRole`)
- `Program.cs` JWT auth configuration — only adding authorization policies alongside what exists
- Existing services (`ITransferService`, `IBillPaymentService`, etc.) — unchanged
- Existing entities (`User`, `Account`, `Transaction`, etc.) — unchanged
- Database connection, Redis, S3, and all existing infrastructure — unchanged

---

## 15. Summary Checklist (For When Implementation Starts)

- [ ] Add `CorporateRole` enum to `Riverly.Domain/Enums/`
- [ ] Add `Organization`, `CorporateUser`, `ApprovalRequest`, `AuditEntry`, `BulkTransferBatch`, `BulkTransferLine` entities
- [ ] Add `DbSet` entries to `ApplicationDbContext`
- [ ] Run EF migration
- [ ] Add `GetOrganizationId()` and `GetCorporateRole()` helpers to `BaseController`
- [ ] Add RBAC policies to `Program.cs`
- [ ] Create DTOs under `Riverly.Application/DTOs/Corporate/`
- [ ] Create service interfaces in `Riverly.Application/Interfaces/`
- [ ] Implement services in `Riverly.Infrastructure/Services/`
- [ ] Register services in `Program.cs`
- [ ] Create controllers in `Riverly.Api/Controllers/Corporate/`
- [ ] Test that `api/v1/` retail endpoints are not affected
