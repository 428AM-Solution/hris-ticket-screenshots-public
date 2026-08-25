# 🐛 E2E Test Report — Issue #467: Cross-Company Approver Leak in Employee Create/Edit

**Issue:** [#467 — Employee Create/Edit: First Approver and Second Approver Must Be Limited to Employees of the Selected Company](https://github.com/428AM-Solution/hris-web/issues/467)
**Reported by:** 428am
**Assigned to:** 428am
**State:** `open` (correctly)
**Test date:** 2026-08-25
**Tester:** 428HR Engineer (Hermes) — Senior Software Tester role
**Verdict:** ❌ **FAIL — Confirmed: approver dropdowns leak ALL employees across all companies/tenants; backend validation is missing; creating/editing employees with cross-company approvers works.**

---

## 1. Executive Summary

This is a **multi-layered bug**:

| Layer | Severity | Status |
|-------|----------|--------|
| **A. Frontend dropdown** (AC1, AC2, AC5, AC6, AC7, AC8, AC13) | 🔴 High | ❌ Leaks all employees regardless of selected Company |
| **B. Company-change refresh** (AC3, AC12) | 🟠 Medium | ❌ Dropdown does not refresh when Company changes |
| **C. Before-selection state** | 🟡 Low | ❌ Dropdown enabled with all options before any Company is selected |
| **D. Backend validation** (AC9, AC10) | 🔴 **Critical** | ❌ Server accepts any `first_approver_id` / `second_approver_id` value, regardless of Company |
| **E. Edit-page persistence** | 🟠 Medium | ⚠️ Cross-company approver persisted to DB after creation |
| **F. Tenant isolation** (AC7, AC8) | 🔴 High | ❌ Tenant users see all employees from all tenants in their approver dropdown |

The **backend bypass (Layer D)** is the most critical issue — it means **frontend filtering alone is insufficient** (as the ticket explicitly warned), and the system stores invalid cross-company approver relationships in the database.

---

## 2. AC-by-AC Test Results

### AC1 — First Approver Is Company-Scoped

> "The First Approver dropdown only displays employees belonging to the employee's selected Company."

**Live test (super_admin, /employees/create):**

```text
=== first_approver_id ===
  Total options: 104
    [] -- None --
    [1] Admin, Super
    [133] AlphaOne, Trial
    [135] AlphaThree, Trial
    [134] AlphaTwo, Trial
    [57] Aquino, Elijah
    [97] Aquino, Kim
    [77] Aquino, Rafael
    [54] Bautista, Christine
    [94] Bautista, Harold
    [74] Bautista, Nicole
    [3] Capa, Angel Nino
    [56] Castillo, Diana
    [96] Castillo, Jason
    ... and 90 more
```

Selecting Company = "428AM" (CompanyID 1) does NOT filter the dropdown — it still shows **all 104 active employees** across all companies and all tenants.

**AC1: ❌ FAIL** — 104 employees shown when ~8 should be shown for CompanyID 1

### AC2 — Second Approver Is Company-Scoped

Identical to AC1 — `second_approver_id` shows exactly the same 104 employees (same list, same length).

**AC2: ❌ FAIL**

### AC3 — Create Page: Company change refreshes approvers

> "On `/employees/create`, changing the Company refreshes both approver dropdowns."

**Live test:**

| Step | first_approver_id count |
|------|------------------------|
| Initial (no Company selected) | 105 |
| Selected Company 1 (428AM) | 105 (unchanged) |
| Changed Company to ACME PWEDE NA | 105 (unchanged) |

Changing the Company dropdown does **not** refetch or filter the approver dropdowns.

**AC3: ❌ FAIL**

### AC4 — Edit Page: approvers load for current Company

> "On the Employee Edit page, both approver dropdowns load employees belonging to the employee's current Company."

**Live test (edited employee in company "Inday Company"):**

```text
=== first_approver_id ===
  Total: 104 (should be ~3-8 employees in Inday Company)
  Selected: -- None --  ← no employee has been saved as approver for this employee yet
```

The edit page populates the dropdown with the **same 104 employees** as the create page — no Company-based filtering.

**AC4: ❌ FAIL**

### AC5 — Company change removes previous Company employees

> "When Company changes during Create/Edit, employees from the previous Company are removed from the approver options."

Already covered by AC3 (the dropdown doesn't refresh). Additionally, even if the user had selected an approver and then changed Company, the previously selected approver would NOT be cleared.

**AC5: ❌ FAIL** (because the dropdown never changes, no removal happens)

### AC6 — Super Admin behavior

> "Super Admin can select from employees of the selected Company, regardless of the Super Admin's own tenant/company context."

Super Admin (`admin@428am.com`) was the test user. The dropdown shows **104 employees from ALL companies** — including ones from other tenants. This violates AC6 (dropdown must contain only selected Company's employees) AND AC8 (no cross-tenant employees).

**AC6: ❌ FAIL** — super_admin sees ALL companies' employees

### AC7 — Tenant User behavior

> "Tenant users can only see approver employees that belong to the selected Company and are within their authorized Tenant."

**Live test (demo@gmail.com, tenant user, /employees/create):**

| Field | Options count | Expected |
|-------|---------------|----------|
| company_id | 14 | (correct — scoped to user's tenant) |
| supervisor | 4 | (correct — scoped to user's tenant) |
| **first_approver_id** | **105** | Should be only employees in selected Company within user's tenant |
| **second_approver_id** | **105** | Same |

The user's `company_id` is properly scoped (14 companies in their tenant) but the `first_approver_id` and `second_approver_id` dropdowns show **ALL 105 employees across ALL tenants** — including employees from tenants the user shouldn't even be aware of.

**AC7: ❌ FAIL** — tenant isolation completely broken on approver dropdowns

### AC8 — Tenant Isolation

> "Employees belonging to another Tenant must never appear as First Approver or Second Approver."

Test employee 133 (AlphaOne Trial) was created with TenantID = 3, while demo@gmail.com's tenant = 1. The dropdown shows employee 133 to demo@gmail.com — **cross-tenant leak**.

**AC8: ❌ FAIL** — confirmed cross-tenant leak

### AC9 — Backend Validation

> "The backend must reject an invalid First Approver or Second Approver if the employee does not belong to the same Company as the employee being created/edited."

**Live security test:**

```
POST /employees (as super_admin)
  company_id: 1 (428AM)
  first_approver_id: 133 (AlphaOne Trial — different company)
  second_approver_id: 133

Response: 200 OK
Body: {"success":true,"message":"Employee created successfully.","redirect_url":"/employees/1d92aeb1-2975-4f58-9b0f-e8bbd17cb2f0"}
```

The server **accepted** the cross-company approver and persisted it to the database. **No backend validation exists.**

**AC9: ❌ FAIL** — backend accepts cross-company approvers (CRITICAL)

### AC10 — No Frontend Bypass

> "Manually submitting an employee ID belonging to another Company must not bypass the restriction."

Same as AC9 — manual POST with cross-company approver IDs succeeds. **The frontend dropdown is the only barrier** and it's trivially bypassable.

**AC10: ❌ FAIL** — confirmed bypass via direct POST

### AC11 — Existing Approver Preservation

> "When editing an employee, preserve the existing First Approver and Second Approver selections if they are still valid employees of the employee's Company."

For the test employee created in AC9, the edit page shows `first_approver_id=133` selected. Since 133 is from a different company, the system should have either:
- Rejected it at create time (didn't)
- Cleared it on edit (didn't)
- Kept it because it's still a valid employee (technically yes, but it's NOT in the same Company)

**AC11: ⚠️ PARTIAL** — approver preserved, but the preservation is of an INVALID cross-company approver (introduced by AC9/AC10 bug)

### AC12 — Invalid Selection Cleared After Company Change

> "If the Company changes and the existing approver no longer belongs to the selected Company, the invalid approver selection must be cleared."

Cannot test reliably because the dropdown itself doesn't refresh on Company change (AC3/AC5 fail). The system would NOT clear the invalid approver because no re-validation happens.

**AC12: ❌ FAIL** (downstream effect of AC3/AC5 failure)

### AC13 — No Cross-Company Employees

> "Employees from other Companies must never be displayed in either approver dropdown."

Confirmed across all tests — 104-105 employees shown in dropdowns regardless of which Company is selected for the employee being created/edited.

**AC13: ❌ FAIL**

### AC14 — Existing UI Behavior (Search Dropdown Preserved)

The existing dropdowns use Select2 (searchable). The Select2 widget is preserved in the current build. This AC is about UI preservation, which is **not violated** by the current implementation.

**AC14: ✅ PASS** (vacuously — no fix needed here)

### AC15 — No Regression

Tested the existing flow (no approver changes, basic employee create). The form submission still works for basic fields.

**AC15: ✅ PASS** (no regression in basic form behavior)

### AC Summary

| AC | Description | Result |
|----|-------------|--------|
| AC1 | First Approver Is Company-Scoped | ❌ FAIL |
| AC2 | Second Approver Is Company-Scoped | ❌ FAIL |
| AC3 | Company change refreshes approvers | ❌ FAIL |
| AC4 | Edit page loads company-scoped approvers | ❌ FAIL |
| AC5 | Company change removes old approvers | ❌ FAIL |
| AC6 | Super Admin company-scoped | ❌ FAIL |
| AC7 | Tenant User tenant-scoped | ❌ FAIL |
| AC8 | Tenant isolation | ❌ FAIL |
| AC9 | Backend validation | ❌ FAIL (CRITICAL) |
| AC10 | No frontend bypass | ❌ FAIL |
| AC11 | Existing approver preservation | ⚠️ PARTIAL |
| AC12 | Invalid selection cleared | ❌ FAIL |
| AC13 | No cross-company display | ❌ FAIL |
| AC14 | Existing UI preserved | ✅ PASS |
| AC15 | No regression | ✅ PASS |

**Score: 1.5/15** — Only AC14 and AC15 pass (and AC15 is trivially passing).

---

## 3. Root Cause Analysis

### 3.1 The bug

The `EmployeeController::create()` and `EmployeeController::edit()` methods both call:

```php
$approverList = $this->employeeRepository->getActiveEmployeeList();
```

This passes `$approverList` to the view, which renders it directly:

```php
<!-- views/employees/create.php, line 444-455 -->
<select id="first_approver_id" name="first_approver_id" ...>
    <option value="">-- None --</option>
    <?php if (isset($approverList) && !empty($approverList)): ?>
        <?php foreach ($approverList as $approver): ?>
            <option value="<?= h($approver['EmployeeID']) ?>">
                <?= h($approver['LastName'] . ', ' . $approver['FirstName']) ?>
            </option>
        <?php endforeach; ?>
    <?php endif; ?>
</select>
```

### 3.2 The repository method

`EmployeeRepository::getActiveEmployeeList()` is:

```php
function getActiveEmployeeList()
{
    return $this->queryAll(
        "SELECT EmployeeID, UUID AS EmployeeUUID, EmployeeNumber, FirstName, LastName
         FROM {$this->table}
         WHERE ReferenceTableStatusID = 1
         ORDER BY LastName, FirstName"
    );
}
```

**No company filter. No tenant filter. No employee-of-same-company filter.** Just `SELECT * FROM M_Employee WHERE active = 1`.

### 3.3 The store() / update() methods

```php
// store(), line 516-517
$firstApproverId = !empty($data['first_approver_id'] ?? '') && is_numeric($data['first_approver_id']) ? (int)$data['first_approver_id'] : null;
$secondApproverId = !empty($data['second_approver_id'] ?? '') && is_numeric($data['second_approver_id']) ? (int)$data['second_approver_id'] : null;

// ...
'mappedData' = [
    // ...
    'FirstApproverID' => $firstApproverId,    // <-- accepted as-is
    'SecondApproverID' => $secondApproverId,  // <-- accepted as-is
    // ...
];
```

The approver IDs are passed straight through to the DB insert **without checking whether the approver belongs to the same Company** as the employee being created/edited.

### 3.4 Why was this missed?

The ticket issue body explicitly anticipated this:

> "Frontend filtering alone is NOT sufficient. The backend/API/database query must enforce the same restriction."

> "A user must not be able to manually submit another employee ID as an approver and bypass the Company/Tenant restriction."

This warning was not heeded during the original implementation. The dropdown filtering is a UI convenience, not a security control, but the developer treated it as sufficient.

---

## 4. Live Evidence

### Screenshot 1: Create page (super_admin)

**File:** [`ISSUE467_01_create_page_superadmin.png`](https://raw.githubusercontent.com/428AM-Solution/hris-ticket-screenshots-public/main/issue-467/ISSUE467_01_create_page_superadmin.png)

Shows the `/employees/create` page with the `first_approver_id` and `second_approver_id` dropdowns populated with 104 employees across all companies/tenants.

### Screenshot 2: Edit page (super_admin)

**File:** [`ISSUE467_02_edit_page_superadmin.png`](https://raw.githubusercontent.com/428AM-Solution/hris-ticket-screenshots-public/main/issue-467/ISSUE467_02_edit_page_superadmin.png)

Shows an employee in "Inday Company" being edited, but the approver dropdowns still show ALL 104 employees — including ones from other tenants.

### Screenshot 3: Tenant user view

**File:** [`ISSUE467_03_create_tenant.png`](https://raw.githubusercontent.com/428AM-Solution/hris-ticket-screenshots-public/main/issue-467/ISSUE467_03_create_tenant.png)

Demo@gmail.com (tenant user with access to 14 companies) sees the same 105 employees in their approver dropdown — confirming cross-tenant leak.

### Screenshot 4: After company change

**File:** [`ISSUE467_04_after_company_change.png`](https://raw.githubusercontent.com/428AM-Solution/hris-ticket-screenshots-public/main/issue-467/ISSUE467_04_after_company_change.png)

After selecting 428AM then changing to ACME PWEDE NA, the approver dropdown is unchanged (still 105 options).

---

## 5. Recommended Fix

### 5.1 Backend changes

**Add a new repository method:**

```php
// EmployeeRepository::getActiveEmployeesByCompany($companyId, $tenantId = null)
function getActiveEmployeesByCompany(int $companyId, ?int $tenantId = null)
{
    $sql = "SELECT e.EmployeeID, e.UUID AS EmployeeUUID,
                   e.EmployeeNumber, e.FirstName, e.LastName
            FROM {$this->table} e
            WHERE e.CompanyID = ?
              AND e.ReferenceTableStatusID = 1";
    $params = [$companyId];

    if ($tenantId !== null && $tenantId > 0) {
        $sql .= " AND (e.TenantID = ? OR e.CompanyID IN (
                      SELECT CompanyID FROM M_Company WHERE TenantID = ?))";
        $params[] = $tenantId;
        $params[] = $tenantId;
    }

    $sql .= " ORDER BY e.LastName, e.FirstName";
    return $this->queryAll($sql, $params);
}
```

**Update EmployeeController::create() and edit():**

```php
// Before showing the form
$selectedCompanyId = (int)($_GET['company_id'] ?? 0);
// Pass company_id from query string or POST back to the view

// For initial render, load approvers based on query string company_id if present
if ($selectedCompanyId > 0) {
    $approverList = $this->employeeRepository->getActiveEmployeesByCompany(
        $selectedCompanyId,
        $effectiveTenantId
    );
} else {
    $approverList = [];  // empty until company selected
}
```

**Add a new AJAX endpoint for company change:**

```php
// POST /api/employees/by-company
public function apiGetEmployeesByCompany()
{
    $companyId = (int)$this->getInput('company_id');
    if ($companyId <= 0) {
        $this->json(['error' => 'company_id required'], 400);
        return;
    }
    $scope = $this->resolveTenantScope();
    $effectiveTenantId = $scope['is_super_admin'] ? null : $scope['tenant_id'];

    $employees = $this->employeeRepository->getActiveEmployeesByCompany(
        $companyId,
        $effectiveTenantId
    );
    $this->json(['success' => true, 'employees' => $employees]);
}
```

Add route in `app/core/App.php`:
```php
$this->router->post('api/employees/by-company', ['controller' => 'Employee', 'action' => 'apiGetEmployeesByCompany']);
```

**Add backend validation in store() and update():**

```php
// In store() / update(), after parsing the approver IDs:
if ($firstApproverId !== null) {
    $approver = $this->employeeRepository->findById($firstApproverId);
    if (!$approver || (int)$approver['CompanyID'] !== (int)$data['company_id']) {
        $this->setFlash('error', 'First approver must belong to the same company.', 'error');
        // re-render form with old input
        return $this->view('employees/create', [...]);
    }
    // Tenant check
    if (!$scope['is_super_admin']) {
        $approverTenant = $this->employeeRepository->getTenantId($approver['EmployeeID']);
        if ($approverTenant !== $scope['tenant_id']) {
            $this->setFlash('error', 'First approver is not in your tenant.', 'error');
            return $this->view('employees/create', [...]);
        }
    }
}
// Same for $secondApproverId
```

### 5.2 Frontend changes

**Update create.php / edit.php views:**

The existing Select2 dropdowns just need to fetch data from the new AJAX endpoint on Company change. Existing Select2 search behavior is preserved.

**Add a JS event handler:**

```javascript
// On Company change, refetch approver dropdowns
$('#company_id').on('change', function() {
    const companyId = $(this).val();
    if (!companyId) {
        // Clear both approver dropdowns
        $('#first_approver_id, #second_approver_id').empty()
            .append('<option value="">-- None --</option>')
            .trigger('change');
        return;
    }
    $.post('/api/employees/by-company', { company_id: companyId }, function(resp) {
        if (resp.success) {
            // Rebuild both dropdowns
            ['first_approver_id', 'second_approver_id'].forEach(function(name) {
                const $sel = $('select[name="' + name + '"]');
                $sel.empty().append('<option value="">-- None --</option>');
                resp.employees.forEach(function(emp) {
                    const label = emp.LastName + ', ' + emp.FirstName;
                    $sel.append('<option value="' + emp.EmployeeID + '">' + label + '</option>');
                });
                $sel.trigger('change');
            });
        }
    });
});
```

**Initial render on edit page:**

```php
// In edit(), load approvers for the employee's current company
$approverList = $this->employeeRepository->getActiveEmployeesByCompany(
    $employee['CompanyID'],
    $effectiveTenantId
);
```

**Before Company selection:**

```php
// In create() initial render
$approverList = [];  // empty dropdown
```

---

## 6. Cross-Cutting Concerns

### 6.1 Test employee cleanup

The cross-company test employee I created (`test.cross.audit@example.test` UUID `1d92aeb1-2975-4f58-9b0f-e8bbd17cb2f0`) needs to be cleaned up:

- First Approver = employee 133 (AlphaOne Trial, company != 1)
- Second Approver = employee 133
- Company = 1 (428AM)

This record demonstrates the bug. Once the fix is deployed, this employee can be cleaned up via the Employees list (admin user).

### 6.2 OWASP Top 10

- **A01 Broken Access Control**: PASS (FAIL) — exactly this bug. Approvers from any tenant/company can be assigned to any employee, bypassing tenant isolation.
- **A03 Injection**: N/A — no SQL changes needed
- **A04 Insecure Design**: FAIL — frontend-only filtering is explicitly called out as insecure by the ticket

---

## 7. File References

| File | Current state |
|------|---------------|
| `app/controllers/EmployeeController.php` | Calls `getActiveEmployeeList()` in both `create()` and `edit()` |
| `app/repositories/EmployeeRepository.php` | `getActiveEmployeeList()` has NO company/tenant filter (root cause) |
| `app/views/employees/create.php` | Lines 444-490 — renders `$approverList` without filtering |
| `app/views/employees/edit.php` | Lines 465-500 — same issue |
| `app/core/App.php` | No AJAX endpoint for company-scoped employee fetch |

---

## 8. Comparison with sibling fields

Interestingly, **`supervisor` dropdown IS correctly scoped** to tenant (4 options for tenant user, all from same tenant). Looking at the source, `getActiveManagersForDropdown()` filters by `PositionTitle LIKE '%Manager%'`, but it doesn't filter by company either. The reason it appears correctly scoped is purely **coincidental** — only 4 employees in the entire system have a Manager/Supervisor/etc. title.

This means **all three** employee-list dropdowns (`supervisor`, `first_approver_id`, `second_approver_id`) leak cross-company employees. The bug is just less obvious for `supervisor` due to data coincidence.

---

## 9. Acceptance Test Plan for the Fix

Once the fix is implemented, verify with these test cases:

1. **AC1 test**: Login as admin, go to /employees/create, select Company "428AM". Count options in `first_approver_id` dropdown. Expected: should be ≤ 8 employees (those who work at 428AM).
2. **AC7 test**: Login as `demo@gmail.com` (tenant user), go to /employees/create, select a Company in their tenant. Count options. Expected: ≤ employees in that company within their tenant.
3. **AC9 test**: Direct POST with `first_approver_id` from a different company → server returns error, no employee record created.
4. **AC3 test**: Change Company dropdown → `first_approver_id` and `second_approver_id` populate with new company's employees.
5. **AC5 test**: Select approver, change Company → approver selection cleared (or kept only if still valid).
6. **AC8 test**: For tenant user, dropdown options should NOT contain employees from other tenants.

---

## 10. Test Artifacts

| Item | Location |
|------|----------|
| Create page screenshot | https://raw.githubusercontent.com/428AM-Solution/hris-ticket-screenshots-public/main/issue-467/ISSUE467_01_create_page_superadmin.png |
| Edit page screenshot | https://raw.githubusercontent.com/428AM-Solution/hris-ticket-screenshots-public/main/issue-467/ISSUE467_02_edit_page_superadmin.png |
| Tenant user screenshot | https://raw.githubusercontent.com/428AM-Solution/hris-ticket-screenshots-public/main/issue-467/ISSUE467_03_create_tenant.png |
| After company change | https://raw.githubusercontent.com/428AM-Solution/hris-ticket-screenshots-public/main/issue-467/ISSUE467_04_after_company_change.png |
| Cross-company POST body | `/tmp/issue467_security5.py` |

---

## 11. Conclusion

Issue #467 exposes a **multi-layer security and UX bug**:

1. **Critical**: Backend has no approver validation — any cross-company approver can be set via POST
2. **High**: Approver dropdowns leak all employees across all companies/tenants to every user
3. **High**: Tenant isolation is broken on approver selection
4. **Medium**: Company change doesn't refresh approver dropdowns

The ticket scope is correctly defined. The fix requires:

- New repository method: `getActiveEmployeesByCompany($companyId, $tenantId)` 
- New AJAX endpoint for dynamic dropdown refresh
- New backend validation in `store()` and `update()` methods
- JS handler for Company change to refetch dropdowns
- Edit page: load approvers for employee's current Company

**Recommended priority**: Critical (backend validation should be fixed first, before the frontend UI).

---

**Test conducted by:** 428HR Engineer
**Test duration:** ~45 minutes (10 test scripts + source code analysis)
**Result:** ❌ **Issue #467 is NOT fixed. The bug is confirmed at multiple layers. Fix recommended as outlined in section 5.**
