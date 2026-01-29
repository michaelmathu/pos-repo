# Transaction Approval (Biometric Approval) Flow – Codebase Guide

This document traces the full transaction approval lifecycle, identifies services/viewmodels/UI, and maps approval triggers to POS actions. No changes are proposed.

---

## 1. Full Transaction Approval Lifecycle

### 1.1 When Approval Is Required

Approval is required when **policy** says so. The decision is made in **one place**:  
`BiometricTransactionService.IsBiometricApprovalRequiredAsync(transactionId, userId)`.

**Logic (all async):**

1. **Transaction and user lookup**  
   Uses `ITransactionService.GetTransactionAsync(transactionId)` and `IUserService.GetUserAsync(userId)`.  
   In WinUI3 these are satisfied by **BiometricTransactionServiceAdapter** and **BiometricUserServiceAdapter**, which hold in-memory transaction/user data stored by the caller **before** starting the workflow.

2. **Policy**  
   `GetTransactionBiometricPolicyAsync(tenantId)` loads policy from `ILocalDatabase.GetBiometricTransactionPolicyAsync(tenantId)`.  
   If no policy exists, a **default policy** is used (e.g. `AmountThreshold = 1000.00m`, `RequiredRoles = ["Manager", "Supervisor"]`, `RequireForTransactionTypes = ["Refund", "Void", "DiscountOverride"]`).

3. **Requirement checks (any one true ⇒ approval required):**
   - **Amount:** `transaction.TotalAmount >= policy.AmountThreshold`
   - **Role:** `user.Role` is in `policy.RequiredRoles`
   - **Transaction type:** `transaction.Type` is in `policy.RequireForTransactionTypes`
   - **Risk:** `IsHighRiskTransactionAsync(transaction, userId)` (e.g. amount above `AmountThreshold * HighRiskAmountMultiplier` if risk-based evaluation is enabled)

**Files / classes:**

| Step | File | Class | Method |
|------|------|-------|--------|
| Requirement check | `SalesLifePOS.Biometric/Services/BiometricTransactionService.cs` | `BiometricTransactionService` | `IsBiometricApprovalRequiredAsync` |
| Policy (default) | Same | Same | `GetTransactionBiometricPolicyAsync` |
| Policy storage | `SalesLifePOS.Biometric/Data/ILocalDatabase.cs` | `ILocalDatabase` | `GetBiometricTransactionPolicyAsync` |
| Risk check | `BiometricTransactionService.cs` | Same | `IsHighRiskTransactionAsync` (private) |

**Sync vs async:** All of the above is **async** (database and policy calls).

**Failure/cancel:** If transaction or user is null, the method returns **false** (approval not required). Policy/risk errors are handled so that missing data or evaluation errors do not block the transaction (e.g. risk evaluation fails safe and returns false).

---

### 1.2 How Biometric Approval Is Triggered

Triggering is **transaction-level only**: the UI (ViewModel or View code) decides that a given action (e.g. Hold, Complete, Refund) needs approval, then:

1. Builds a **Transaction** (id, tenant, total amount, type) and **User** (id, role).
2. **Stores** them in the adapters so the workflow can resolve them by id.
3. Resolves **BiometricTransactionWorkflow** from DI and calls **ProcessTransactionWithBiometricApprovalAsync(transaction, user)**.

**Flow inside the workflow (all async):**

1. **Is approval required?**  
   `_biometricService.IsBiometricApprovalRequiredAsync(transaction.Id, user.Id)`.  
   If false → return `TransactionApprovalResult` with `Status = NotRequired` and stop.

2. **Enrolled biometric types**  
   `GetUserEnrolledBiometricTypesAsync(user.Id)` via `ILocalDatabase.GetBiometricEnrollmentsForUserAsync`.  
   If the list is empty, **Facial (Windows Hello)** is used as default (no pre-enrollment needed).

3. **Prompt + authenticate (up to 3 attempts):**
   - **Show prompt:** `_notificationService.ShowBiometricPromptAsync(user.Id, BiometricPromptRequest)`  
     → WinUI3 shows a **ContentDialog** with **BiometricApprovalDialog** (message, amount, Verify/Cancel).
   - **Authenticate:** `_biometricService.AuthenticateTransactionAsync(transaction.Id, user.Id, biometricType)`  
     → Device selection, connect, capture, (for non-Facial) verify against enrolled template; for **Facial**, success of capture is treated as verification.

4. **Result:**
   - Success → workflow builds result with `Status = Approved`, records approval (local DB + transaction service + security log), returns.
   - Failure after all attempts → **ProcessAlternativeApprovalAsync** (PIN, password, manager override, Duo).  
     Current implementation of alternatives is largely **stub** (e.g. PIN/password not fully implemented).  
     If no alternative succeeds → `Status = Rejected`.

**Files / classes:**

| Step | File | Class | Method |
|------|------|-------|--------|
| Entry | `SalesLifePOS.Biometric/Workflows/BiometricTransactionWorkflow.cs` | `BiometricTransactionWorkflow` | `ProcessTransactionWithBiometricApprovalAsync` |
| Enrolled types | Same | Same | `GetUserEnrolledBiometricTypesAsync` (uses `ILocalDatabase`) |
| Prompt | Same | Same | `ShowBiometricPromptAsync` |
| Authenticate | `SalesLifePOS.Biometric/Services/BiometricTransactionService.cs` | `BiometricTransactionService` | `AuthenticateTransactionAsync` |
| Fallback | `BiometricTransactionWorkflow.cs` | Same | `ProcessAlternativeApprovalAsync`, `TryPINApprovalAsync`, etc. |

**Sync vs async:** Entire flow is **async**.  
**Cancel:** The dialog’s **Cancel** closes the prompt and completes `ShowBiometricPromptAsync`; the workflow **does not** use the dialog result and **always** calls `AuthenticateTransactionAsync` next. So cancelling the dialog does not skip the biometric attempt; cancellation at the **device** (e.g. Windows Hello) or failure/timeout is what leads to retry or rejection.

---

### 1.3 How Approval Results Are Returned

- **Workflow** returns **TransactionApprovalResult**:
  - `Status`: `NotRequired`, `Approved`, `Rejected`, `Failed`, etc.
  - `ApprovalMethod`: `Biometric`, `ManagerOverride`, `PIN`, etc.
  - `BiometricType`, `ConfidenceScore`, `DeviceId`, `ApprovalId` when approved.
  - `ErrorMessage`, `BiometricErrorMessage` on failure/rejection.

- **Callers** (e.g. CashSaleViewModel, RefundCashSaleView):
  - Await `ProcessTransactionWithBiometricApprovalAsync(...)`.
  - If `Status` is **Approved** or **NotRequired** → continue (save/complete/post).
  - Otherwise → show error (e.g. “Transaction approval was rejected or failed”) and **abort** the POS action (no save/complete).

**Files:**

| Caller | File | How result is used |
|--------|------|---------------------|
| CashSaleViewModel | `SalesLifePOSWinUI3/ViewModel/CashSale/CashSaleViewModel.cs` | `CheckBiometricApprovalAsync` returns `bool`; if false, method returns without saving. |
| RefundCashSaleView | `SalesLifePOSWinUI3/Views/RefundCashSale/RefundCashSaleView.xaml.cs` | Checks `approvalResult.Status`; if not Approved/NotRequired, shows dialog and returns without calling refund service. |

---

### 1.4 How Approval Is Logged

Approval is recorded in **three** places (all from **BiometricTransactionService.RecordBiometricApprovalAsync**, or from the workflow’s **RecordBiometricApprovalAsync** plus transaction service):

1. **Local DB (approval record)**  
   `ILocalDatabase.SaveBiometricApprovalAsync(BiometricApproval)`  
   - WinUI3: `BiometricLocalDatabase` (in-memory) or SQLite impl.  
   - Stores: Id, TransactionId, UserId, BiometricType, Status, ApprovalTime, ConfidenceScore, DeviceId.

2. **Transaction linkage**  
   `ITransactionService.AddBiometricApprovalAsync(transactionId, approvalId)`  
   - WinUI3: **BiometricTransactionServiceAdapter** – currently a no-op (no server persistence of approval id in the snippet).

3. **Security / audit**  
   `ISecurityService.LogBiometricApprovalAsync(userId, transactionId, type, score)`  
   - WinUI3: **BiometricSecurityService** builds a **BiometricAuditEvent** (e.g. VerificationSucceeded) and stores it via **StoreAuditEventAsync** (e.g. local audit store).

**Files:**

| Purpose | File | Class | Method |
|---------|------|-------|--------|
| Approval record | `SalesLifePOS.Biometric/Services/BiometricTransactionService.cs` | `BiometricTransactionService` | `RecordBiometricApprovalAsync` |
| Approval record | `SalesLifePOS.Biometric/Workflows/BiometricTransactionWorkflow.cs` | `BiometricTransactionWorkflow` | `RecordBiometricApprovalAsync` (also creates entity and saves to local DB) |
| Local DB | `SalesLifePOS.Biometric/Data/ILocalDatabase.cs` | `ILocalDatabase` | `SaveBiometricApprovalAsync` |
| Security log | `SalesLifePOSWinUI3/Services/Biometric/BiometricSecurityService.cs` | `BiometricSecurityService` | `LogBiometricApprovalAsync` |

---

## 2. Services, ViewModels, UI, and Biometric Types

### 2.1 Services

| Service | Role | File |
|---------|------|------|
| **BiometricTransactionService** | Policy check, device capture/verify, approval recording, Windows Hello handling for Facial | `SalesLifePOS.Biometric/Services/BiometricTransactionService.cs` |
| **BiometricTransactionWorkflow** | Orchestrates: required? → enrolled types → prompt → authenticate → record or fallback | `SalesLifePOS.Biometric/Workflows/BiometricTransactionWorkflow.cs` |
| **BiometricTransactionServiceAdapter** | Implements `ITransactionService`: GetTransactionAsync (in-memory), AddBiometricApprovalAsync (no-op in snippet) | `SalesLifePOSWinUI3/Services/Biometric/BiometricTransactionServiceAdapter.cs` |
| **BiometricUserServiceAdapter** | Implements `IUserService`: GetUserAsync (in-memory user stored by caller) | WinUI3 Services/Biometric (same area as adapter) |
| **BiometricNotificationService** | Implements workflow’s **INotificationService**: shows ContentDialog with BiometricApprovalDialog | `SalesLifePOSWinUI3/Services/BiometricNotificationService.cs` |
| **BiometricDeviceManager** | Resolves device by BiometricType (Fingerprint, Facial, etc.), caches, initializes | `SalesLifePOS.Biometric/Devices/BiometricDeviceManager.cs` |
| **WindowsBiometricService** | Implements **IBiometricDevice** for Fingerprint and Facial (Windows Hello); CaptureAsync branches to CaptureWindowsHelloAsync or CaptureWbfFingerprintAsync | `SalesLifePOS.Biometric/Platforms/Windows/WindowsBiometricService.cs` |
| **ILocalDatabase** | Policy, approvals, enrollments (SQLite or in-memory impl in WinUI3) | `SalesLifePOS.Biometric/Data/ILocalDatabase.cs` |
| **ISecurityService** | Audit logging (e.g. LogBiometricApprovalAsync) | `SalesLifePOS.Biometric/Security/ISecurityService.cs`, impl in WinUI3 |

Policy is **BiometricTransactionPolicy** / **BiometricPolicy** (amount threshold, roles, transaction types, risk settings, fallback flags). Stored via **ILocalDatabase**; default used if none exists.

### 2.2 ViewModels

| ViewModel | Role | File |
|-----------|------|------|
| **BiometricApprovalViewModel** | Binds to BiometricApprovalDialog: transaction number, amount, message, Verify/Cancel/Use PIN/Request Manager. **OnCloseRequested** is invoked for any of those actions; the dialog does not distinguish Verify vs Cancel for the workflow (workflow ignores dialog result and always runs AuthenticateTransactionAsync). | `SalesLifePOSWinUI3/ViewModel/Biometric/BiometricApprovalViewModel.cs` |
| **CashSaleViewModel** | Holds **CheckBiometricApprovalAsync**, **CreateTransactionForBiometricApproval**, **GetCurrentUserForBiometricApproval**; calls them before Hold, Complete, Invoice complete, Hold-to-complete, Transaction complete, Sales order complete. | `SalesLifePOSWinUI3/ViewModel/CashSale/CashSaleViewModel.cs` |

### 2.3 UI Components

| Component | Role | File |
|-----------|------|------|
| **BiometricApprovalDialog** | UserControl shown inside the approval ContentDialog; displays message, amount, Verify/Cancel (and optionally Use PIN, Request Manager). | `SalesLifePOSWinUI3/Views/Biometric/BiometricApprovalDialog.xaml` + `.xaml.cs` |
| **ContentDialog** (in BiometricNotificationService) | Wraps BiometricApprovalDialog; Primary = “Verify”, Secondary = “Cancel”. Result (Primary vs Secondary) is not used by the workflow. | `SalesLifePOSWinUI3/Services/BiometricNotificationService.cs` |

### 2.4 Windows Hello and Other Biometric Types

- **BiometricType** enum: None, Fingerprint, Facial, Iris, Voice, Palm.
- **Facial** is used for **Windows Hello**: no pre-enrolled template in app; Windows manages enrollment. In **BiometricTransactionService.AuthenticateTransactionAsync**:
  - If `type == BiometricType.Facial`, template lookup is skipped.
  - After capture success, verification is considered done; approval is recorded with capture quality score.
- **BiometricDeviceManager.GetDeviceForTypeAsync(BiometricType)** returns the first registered device that supports that type (e.g. **WindowsBiometricService** supports Fingerprint | Facial).
- **WindowsBiometricService**: CaptureAsync dispatches to **CaptureWindowsHelloAsync** (Facial) or **CaptureWbfFingerprintAsync** (fingerprint). Current code uses placeholder/mock WinRT usage (e.g. UserConsentVerifier commented); real Windows Hello would use `Windows.Security.Credentials.UI.UserConsentVerifier.RequestVerificationAsync`.

---

## 3. Step-by-Step Summary (with Files, Methods, Sync vs Async, Failure/Cancel)

| Step | Summary | Files / classes | Key methods | Sync/async | On failure / cancel |
|------|---------|-----------------|-------------|------------|---------------------|
| 1. POS action (e.g. Hold/Complete) | ViewModel builds Transaction + User, stores in adapters, calls workflow | CashSaleViewModel, RefundCashSaleView | CreateTransactionForBiometricApproval, GetCurrentUserForBiometricApproval, CheckBiometricApprovalAsync; adapter.StoreTransaction/StoreUser | Async | If CheckBiometricApprovalAsync returns false: show message, return without saving. On exception in CheckBiometricApprovalAsync, current code returns true (allow) to avoid breaking flow. |
| 2. Is approval required? | Load transaction/user from adapters, load policy, check amount/role/type/risk | BiometricTransactionService, ILocalDatabase | IsBiometricApprovalRequiredAsync, GetTransactionBiometricPolicyAsync, IsHighRiskTransactionAsync | Async | Transaction/user null → false. Policy/risk errors → fail safe (e.g. false). |
| 3. Enrolled types | Get user enrollments; if none, use Facial (Windows Hello) | BiometricTransactionWorkflow, ILocalDatabase | GetUserEnrolledBiometricTypesAsync | Async | Empty list → Facial only. |
| 4. Show prompt | Show ContentDialog with BiometricApprovalDialog on UI thread | BiometricNotificationService | ShowBiometricPromptAsync, ShowDialogAsync | Async (UI dispatch) | Window/XamlRoot null → complete with false. Dialog Cancel/Primary both lead to workflow continuing to step 5 (workflow does not use result). |
| 5. Authenticate | Get device, connect, capture, (non-Facial) verify; Facial: capture success = verified | BiometricTransactionService, BiometricDeviceManager, IBiometricDevice (e.g. WindowsBiometricService) | AuthenticateTransactionAsync, GetDeviceForTypeAsync, CaptureAsync, RecordBiometricApprovalAsync | Async | Not required → NotRequired. Capture/verify fail → retry (up to 3) or return Failed. Device errors → BiometricAuthResult.Failed. |
| 6. Record approval | Save BiometricApproval to local DB, AddBiometricApprovalAsync, LogBiometricApprovalAsync | BiometricTransactionService, Workflow, ILocalDatabase, ITransactionService, ISecurityService | RecordBiometricApprovalAsync, SaveBiometricApprovalAsync, AddBiometricApprovalAsync, LogBiometricApprovalAsync | Async | Logging failures do not change workflow result. |
| 7. Alternative approval | If biometric failed: try PIN, password, manager override, Duo (stubs) | BiometricTransactionWorkflow | ProcessAlternativeApprovalAsync, TryPINApprovalAsync, TryManagerOverrideAsync, etc. | Async | Most alternatives return false; manager override can succeed if user has Manager/Supervisor role. If all fail → Rejected. |
| 8. Return to caller | TransactionApprovalResult (Status, ApprovalMethod, …) | BiometricTransactionWorkflow | ProcessTransactionWithBiometricApprovalAsync return value | Async | Caller checks Approved or NotRequired → proceed; else show error and abort POS action. |

---

## 4. Approval Triggers vs POS Actions

Approval is **transaction-level only**. It runs **before** the corresponding save/complete/post. There are **no** line-level approval triggers (no dedicated approval for “line price change”, “line discount”, or “line delete” in this codebase).

| POS action | When approval runs | File (method) | Transaction type passed |
|------------|--------------------|---------------|---------------------------|
| **Hold** (save transaction with status Hold) | Before SaveCashSale(SaveCashSaleLine) | CashSaleViewModel – **ExecuteBtnHoldCommand** | "CashSale" |
| **Complete current transaction** (payment complete) | Before SaveCashSale + SaveCashSaleLine + SaveCashSalePayment etc. | CashSaleViewModel – **CurrentTransactionComplete** | "CashSale" |
| **Complete held transaction** (update status + lines) | Before UpdateCashSaleStatus + UpdateCashSaleLine | CashSaleViewModel – **HoldTransactionToComplete** | "CashSale" |
| **Transaction complete** (another complete path) | Before SaveCashSale (Status Complete) | CashSaleViewModel – **TransactionToComplete** | "CashSale" |
| **Invoice complete** | Before SaveCashSale (Status Complete) | CashSaleViewModel – **InvoiceToComplete** | "CashSale" |
| **Sales order complete** | Before SaveOrders + SaveOrderItem | CashSaleViewModel – **SalesOrderToComplete** | "SalesOrder" |
| **Refund (new)** | Before SaveCashSaleRefund | RefundCashSaleView – **ButtonNew_Click** | "Refund" |

**Not present in codebase:**

- **Line price change** – no approval gate found.
- **Line discount** – no approval gate found.
- **Line delete** – no approval gate found.
- **Payment** – approval is tied to “complete” flows that include payment, not a separate “payment only” step.
- **Posting** – no separate “posting” approval hook identified; posting may be part of the same complete/save flows above.

Policy can still **require** approval by **transaction type** (e.g. `RequireForTransactionTypes` including "Refund", "Void", "DiscountOverride") or by **amount/role/risk**; the trigger remains “when the ViewModel/View runs the approval check before that transaction-level action.”

---

## 5. Dependency Registration (ServiceLocator)

Relevant registrations in **SalesLifePOSWinUI3/Configuration/ServiceLocator.cs**:

- **ITransactionService** → BiometricTransactionServiceAdapter  
- **IUserService** → BiometricUserServiceAdapter  
- **INotificationService** (workflow) → BiometricNotificationService  
- **ILocalDatabase** → BiometricLocalDatabase (or SQLite)  
- **IBiometricTransactionService** → BiometricTransactionService  
- **BiometricTransactionWorkflow** → scoped  
- **AddBiometricDevices()** → registers **WindowsBiometricService**, **USBScannerService**, **MobileBiometricService** as **IBiometricDevice**, and **BiometricDeviceManager** as **IBiometricDeviceManager**

---

This completes the trace of the existing Transaction Approval (Biometric Approval) flow and where it ties into POS actions.
