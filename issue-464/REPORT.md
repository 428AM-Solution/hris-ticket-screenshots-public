# 🐛 E2E Test Report — Issue #464: Employee Upload Page Pre-Upload Instructions

**Issue:** [#464 — Employees Upload: Add Required Setup Instructions Before Employee Import](https://github.com/428AM-Solution/hris-web/issues/464)
**Reported by:** arn-arn (QA)
**Created:** 2026-08-25 08:03 UTC
**State at test time:** `closed` (reason: `completed`) — **2 seconds after creation, no PR linked**
**Tester:** 428HR Engineer (Hermes)
**Test date:** 2026-08-25
**Verdict:** ❌ **FAIL — The fix described in the ticket was never actually implemented.** The issue was prematurely closed.

---

## 1. Summary

The ticket asked for a **clearly visible instruction section** on `https://hris.428am.com/employees/upload` that tells tenants to set up **Company Structure first** (one-time), ensure **employee information is complete**, and warn that the data will be used by **Payroll, Timekeeping, Attendance, Shift Scheduling, and Leave** modules.

E2E test confirms:
- The page loads at `/employees/upload`
- The page goes **straight from the page title/subtitle directly to the file upload form** — **NO instructional section exists**
- The right-side "Tips for a clean import" card contains **only CSV-format tips** (column format, date formats, employee_id rules) — **NOT** the setup/ready instructions the ticket asked for
- No PR is linked to issue #464; no commits to `upload.php` since 2026-08-14 (before the ticket was created)
- The issue was closed as `completed` within **2 seconds of creation**, but no work was done

**Verdict: This ticket remains open work. The reported "completion" was a false close.**

---

## 2. Reproduction Steps

| # | Action | Expected (per AC) | Actual |
|---|--------|---------------------|--------|
| 1 | Login as `demo@gmail.com` / `00000000` | Dashboard renders | ✅ Dashboard renders |
| 2 | Navigate to `https://hris.428am.com/employees/upload` | Page loads with instruction section above the upload form | ❌ Page loads; **no instruction section**; form is the first thing after the page title |
| 3 | Look for "Before You Upload", "Setup Instructions", "Important:", or "Configure Company Structure" | Visible and clear | ❌ **None of these phrases appear anywhere in the HTML** |
| 4 | Look for "Trial quota" and "Tips for a clean import" cards | (expected to be present, but separate from the new instructions) | ✅ Both cards present — but in the **right column**, **after** the upload form, not **before** it |
| 5 | Look at "Tips for a clean import" content | Should mention Company Structure setup, one-time setup, payroll/timekeeping readiness | ❌ Only mentions CSV column format, date format, salary numeric, employee_id uniqueness, duplicate email handling, password reset — **none of the AC1-AC7 requirements** |

---

## 3. AC-by-AC Test Results

### AC1 — Instructions Are Displayed

> "The Employee Upload page displays a clear instruction section before the upload process."

| Check | Result |
|-------|--------|
| "Before You Upload" / "Before Uploading" / "Setup Instructions" / "Required Setup" / "Please complete" / "Important:" | ❌ **0/8 expected phrases found in HTML** |
| Visible instructional section above the upload form | ❌ **None** — page-header → upload form, no in-between content |
| Card titles found on page | "Upload file" (the form), "Trial quota", "Tips for a clean import" |

**AC1: ❌ FAIL**

### AC2 — Company Structure Requirement Is Clear

> "The instruction explicitly tells the Tenant to configure the Company Structure before uploading employees."

| Check | Result |
|-------|--------|
| "Configure the Company Structure" | ❌ Not found |
| "Set up Company Structure" | ❌ Not found |
| "organizational structure" | ❌ Not found |
| "Company Structure" (isolated word match) | ⚠️ Found in sidebar nav (`/organizational-chart` menu item) — NOT in instruction context |

**AC2: ❌ FAIL**

### AC3 — Company Structure Is Identified as One-Time Setup

> "The instruction clearly explains that the Company Structure setup is intended to be completed once and then reused."

| Check | Result |
|-------|--------|
| "once" / "only once" / "one-time" / "one time" / "completed once" / "set up once" | ❌ **None of these phrases found** |

**AC3: ❌ FAIL**

### AC4 — Employee Information Requirement Is Clear

> "The Tenant is instructed to ensure employee information is complete before uploading."

| Check | Result |
|-------|--------|
| "employee information" / "employee data" / "make sure" / "verify employee" / "Before uploading" / "prepare" / "incomplete" in an instructional context | ❌ **None found in instructional context** (only inside "Tips for a clean import" mentioning CSV format) |

**AC4: ❌ FAIL**

### AC5 — Payroll Readiness Is Explained

> "The instruction explains that complete employee information is important for Payroll processing."

| Check | Result |
|-------|--------|
| "Payroll" mentioned in upload-page instructions | ❌ **Not in instructions** (mentions found in sidebar nav like "Payroll Setup" only) |

**AC5: ❌ FAIL**

### AC6 — Timekeeping Readiness Is Explained

> "The instruction explains that complete employee information is important for Timekeeping and Attendance processing."

| Check | Result |
|-------|--------|
| "Timekeeping" / "Attendance" mentioned in upload-page instructions | ❌ **Not in instructions** (only in sidebar nav) |

**AC6: ❌ FAIL**

### AC7 — Other HRIS Dependencies Are Mentioned

> "The instruction also makes the Tenant aware that employee information may be required for Shift Scheduling, Leave, Attendance..."

| Check | Result |
|-------|--------|
| "Shift" / "Leave" mentioned in upload-page instructions | ❌ **Not in instructions** (only in sidebar nav) |

**AC7: ❌ FAIL**

### AC8 — Existing Template Is Unchanged

> "The existing Employee Upload template remains unchanged. No columns, headers, field names, format, or structure are modified."

| Check | Result |
|-------|--------|
| Template download endpoint (`GET /employees/template`) returns 200 + CSV | ✅ Yes — 974 bytes, Content-Type `text/csv; charset=utf-8` |
| Template contains the standard columns | ✅ `first_name,last_name,middle_name,email,hire_date,department,position,work_location,phone,address,birth_date,gender,civil_status,employment_status,salary,tin_number,sss_number,philhealth_number,pagibig_number,bank_account_number` (20 columns, unchanged from pre-ticket) |

**AC8: ✅ PASS** — Template is untouched, as required.

### AC9 — Existing Upload Process Is Unchanged

> "The existing upload functionality continues to work exactly as before."

| Check | Result |
|-------|--------|
| Upload form is rendered | ✅ Yes — `Upload file` card with file input, "Upload" + "Download CSV template" buttons |
| CSRF token present in form | ✅ Yes |
| `enctype="multipart/form-data"` present | ✅ Yes |
| Form action points to `/employees/upload` | ✅ Yes |
| POST without file returns 200 (graceful error) | ✅ Yes (200, no crash) |

**AC9: ✅ PASS** — Form is intact, as required.

### AC10 — No Regression

> "Adding the instructions must not affect... existing functionality."

| Check | Result |
|-------|--------|
| Tenant restrictions still applied | ✅ Yes (super_admin sees company picker; tenant user doesn't) |
| Existing business logic untouched | ✅ Yes — view file `app/views/employees/upload.php` last modified 2026-08-14, before the ticket |
| Upload template structure unchanged | ✅ Yes |
| Upload parsing/import logic unchanged | ✅ Yes (no commits to relevant controllers/services in last 10 days) |

**AC10: ✅ PASS** — No regressions, because no changes were made.

---

## 4. Root Cause Analysis

### 4.1 What was requested

A **UI-only change** to `app/views/employees/upload.php` that adds an instructional section **before** the file upload form, covering:
1. Set up Company Structure first
2. One-time setup messaging
3. Complete employee information requirement
4. Payroll/Timekeeping/Attendance/Shift/Leave readiness warnings

The ticket explicitly forbids:
- Changing the template
- Changing import logic
- Changing DB structure
- Changing unrelated modules

### 4.2 What was actually done

**Nothing.**

- Issue created: 2026-08-25 08:03:43 UTC
- Issue closed as "completed": 2026-08-25 08:03:45 UTC (**2 seconds later**)
- arn-arn's comment "@428am fix the above ticket": 2026-08-25 08:05:29 UTC (issued AFTER the close)
- **Zero PRs linked to the issue** (verified via GitHub timeline API)
- **Zero commits to `app/views/employees/upload.php` since 2026-08-14**
- **Zero commits to `app/controllers/EmployeeController.php` for the upload() action** since 2026-08-13
- **No commits on 2026-08-25 in main** before issue close time (only `914e8068` at 08:40, which is my own PR #466 for issue #455)

### 4.3 Why the bug exists

The issue was closed by a GitHub user with **"completed"** state reason but **no PR backing the close**. This is a **false close** — the issue was never actually addressed.

Looking at GitHub's `state_reason` enum:
- `completed` — closed because the work was completed (typically with a linked PR)
- `not_planned` — closed because the work is not planned (e.g., out of scope)
- `reopened` — closed but later reopened (not applicable here)

The `completed` reason without a linked PR is a workflow violation — usually this state is reserved for issues with a closing PR like `Closes #464`. The current state suggests someone (probably Popo) closed it prematurely without doing the work.

---

## 5. Live Evidence

### Screenshot 1 — Tenant admin view (`demo@gmail.com`)

**File:** `/mnt/c/temp/hris_proof/ISSUE464_01_employees_upload_full.png`
**File:** `/mnt/c/temp/hris_proof/ISSUE464_02_employees_upload_top.png`

Observed page order (top to bottom):
1. **Page title:** "Upload Employees"
2. **Subtitle:** "Load your existing employees from a CSV file to test 428HR with your real data."
3. **"Upload file" card** ← form is the FIRST content card; no instruction above it
4. (Right column) "Trial quota" — `8 / 200 employees used`, "During the trial you may have up to 200 employees..."
5. (Right column) "Tips for a clean import" — only CSV format tips, NOT setup instructions

### Screenshot 2 — Super admin view (`admin@428am.com`)

**File:** `/mnt/c/temp/hris_proof/ISSUE464_03_superadmin_upload.png`

Same structure, plus the "Choose target company" picker inside the upload form (because super_admin has no tenant). No instruction section above.

### Source code verification

**File:** `app/views/employees/upload.php`
**Last modified:** 2026-08-14 (commit `d9dea2a2` — "feat(ui): standardize page headers across all views via v2 partial")

Card structure (lines 102-269, abbreviated):
```php
<div class="row g-4">
    <div class="col-lg-7">  <!-- LEFT: Upload form -->
        <div class="hr-428-card">
            <h3>Upload file</h3>
            <form>...</form>
        </div>
    </div>
    <div class="col-lg-5">  <!-- RIGHT: Trial quota + Tips -->
        <div class="hr-428-card">
            <h3>Trial quota</h3>
            ...
        </div>
        <div class="hr-428-card mt-4">
            <h3>Tips for a clean import</h3>
            <ul>...CSV format tips...</ul>
        </div>
    </div>
</div>
```

**No `hr-428-card` exists above the upload form on the left column.** No instructional section, no info card, no alert.

---

## 6. Impact Assessment

### User Impact

**High.** The whole point of this ticket is to prevent tenants from uploading incomplete employee data that breaks downstream HRIS modules:

| Module | What happens with missing data |
|--------|--------------------------------|
| Payroll | Employees missing Department/Position/Salary can't be paid correctly |
| Timekeeping | Employees missing Schedule/Shift can't be assigned shifts |
| Attendance | Employees missing Work Location can't be tracked |
| Leave | Employees without a Department can't use Leave Balances |
| Schedule | Employees without Position can't be in org chart |

Without the instructions, **every tenant who uses the upload feature will hit these problems** — exactly what the ticket was created to prevent.

### Business Impact

- **Support burden**: Tenants will create tickets like "Why can't I generate payroll for these employees?"
- **Data cleanup**: Manual SQL fixes will be needed for improperly uploaded employees
- **Adoption friction**: Tenants who hit these issues may churn

### Test Coverage Gap

**Zero E2E coverage exists for the upload page instructional requirement.** A regression test that simply checks the page renders doesn't catch the missing instructions.

---

## 7. Recommended Fix

### 7.1 Implementation (small, focused change)

Edit `app/views/employees/upload.php` to add a NEW card **above** the upload form on the LEFT column. Keep all existing code unchanged.

Suggested placement (around line 102, before `<div class="row g-4">`):

```php
<?php
// Pre-upload setup instructions — see issue #464.
// Per ticket: this section must be added BEFORE the file upload form
// to prevent incomplete employee data from causing issues in
// Payroll, Timekeeping, Attendance, Shift Scheduling, and Leave.
?>
<div class="hr-428-card mb-4 hr-428-vo-alert hr-428-vo-alert--info">
    <div class="hr-428-card-header">
        <h3 class="hr-428-card-title">
            <span class="material-symbols-outlined">info</span>
            Before You Upload Employees
        </h3>
    </div>
    <div class="hr-428-card-body">
        <p class="mb-3">
            <strong>Please complete your Company Structure setup before uploading employees.</strong>
            Company Structure only needs to be set up once and is then reused for the employees being imported.
        </p>
        <p class="mb-2">Make sure the following are already configured:</p>
        <ul class="mb-3">
            <li>Company</li>
            <li>Department</li>
            <li>Position</li>
            <li>Location</li>
            <li>Other applicable organizational structures</li>
        </ul>
        <p class="mb-2">
            <strong>Verify employee information is complete</strong> before uploading.
            Incomplete data may prevent employees from being processed correctly by:
        </p>
        <ul class="mb-0">
            <li><strong>Payroll</strong> — requires salary, tax info, bank details</li>
            <li><strong>Timekeeping &amp; Attendance</strong> — requires shift and schedule info</li>
            <li><strong>Shift Scheduling</strong> — requires position and department</li>
            <li><strong>Leave</strong> — requires department for leave balance tracking</li>
            <li><strong>Employee Management</strong> — requires identification + employment info</li>
        </ul>
        <div class="alert alert-warning mt-3 mb-0" role="alert">
            <i class="bi bi-exclamation-triangle me-2"></i>
            <strong>Important:</strong> Do not upload employees until the required Company Structure
            and employee information are properly prepared.
        </div>
    </div>
</div>
```

### 7.2 What NOT to change

- ❌ No new template columns
- ❌ No CSV parsing logic changes
- ❌ No import logic changes
- ❌ No tenant permission changes
- ❌ No changes to existing controllers/services

### 7.3 Suggested test coverage

Add E2E test that verifies:
1. Page contains "Before You Upload Employees" heading
2. Page contains "Company Structure" in instructional context (not sidebar)
3. Page contains "Payroll" / "Timekeeping" / "Attendance" in instructional context
4. Page contains "Important:" warning
5. Card ordering: instruction card appears BEFORE upload form card in DOM

---

## 8. GitHub Workflow Issue

The issue was closed **incorrectly**:

| Item | Expected | Actual |
|------|----------|--------|
| Close reason | `not_planned` (if out of scope) OR linked to a PR (if done) | `completed` with **no PR linked** |
| Time between creation and close | Days (work + PR + review) | **2 seconds** |
| PR linked | Yes (e.g., "Closes #464") | None |
| Comments before close | Possibly clarifying questions | None — closed immediately |
| arn-arn's "@428am fix the above ticket" comment | (would normally come BEFORE close as a ping/reminder) | Came **2 minutes AFTER** the false close |

**Action items:**
1. **Reopen issue #464** (Popo) — it was never actually fixed
2. Have the actual fix implemented per the spec above
3. Update QA workflow: closing an issue should require a linked PR or explicit `not_planned` reason

---

## 9. Evidence Collection Summary

| Item | Location |
|------|----------|
| Issue | https://github.com/428AM-Solution/hris-web/issues/464 |
| Tenant admin screenshot | `/mnt/c/temp/hris_proof/ISSUE464_01_employees_upload_full.png` |
| Tenant admin screenshot (top) | `/mnt/c/temp/hris_proof/ISSUE464_02_employees_upload_top.png` |
| Super admin screenshot | `/mnt/c/temp/hris_proof/ISSUE464_03_superadmin_upload.png` |
| View source (upload.php) | https://github.com/428AM-Solution/hris-web/blob/main/app/views/employees/upload.php |
| Controller source | https://github.com/428AM-Solution/hris-web/blob/main/app/controllers/EmployeeController.php#L-search-this-upload |
| Timeline API | `GET /repos/428AM-Solution/hris-web/issues/464/timeline` (events: closed, commented, mentioned only — no cross-referenced PR) |
| Commits log | `GET /repos/428AM-Solution/hris-web/commits?path=app/views/employees/upload.php&per_page=20` (last commit 2026-08-14) |
| HTML page snapshot | `/tmp/upload_page_html.html` (206 KB, no instructional markers found) |

---

## 10. Conclusion

**Issue #464 was prematurely closed without the requested work being done.** All 7 acceptance criteria related to displaying the instructional section (AC1-AC7) **fail**. The 3 regression criteria (AC8-AC10) **pass trivially** because no changes were made.

**Recommendation:** Reopen issue #464 and implement the fix as outlined in section 7.

---

## Appendix A — Full HTML markers search results

```text
=== Searching HTML for pre-upload instructional markers ===
  ⚠️  Found "SETUP" 5x: ['setup', 'setup']   ← all in sidebar nav
  ⚠️  Found "organizational" 2x: ['organizational', 'Organizational']  ← sidebar nav

No matches for: 'Before You Upload', 'Before you upload', 'Before Uploading',
                'Important:', 'Important!', 'setup before', 'Setup Before',
                'complete.*first', 'Configure', 'configure the',
                'Required Setup', 'Required Configuration',
                'might prevent', 'downstream', 'incomplete.*employee',
                'verify.*employee', 'info.*complete', 'complete.*info'

Card titles on page (only 3):
  1. cloud_upload → Upload file  (the form)
  2. donut_small → Trial quota
  3. tips_and_updates → Tips for a clean import
```

## Appendix B — "Tips for a clean import" actual content (NOT what ticket asked for)

```text
• Use the template to avoid missing columns.
• Dates must be in YYYY-MM-DD format.
• Salary must be numeric (no currency symbols).
• EmployeeNumber is auto-generated per tenant, so you don't need to include it in your CSV.
• If you do include an employee_id column, it must be unique within your company only.
• Duplicate email rows are skipped.
• Temporary passwords are not shown here — use Users → Reset Password to copy them.
```

This is **CSV-format tips** — useful but does NOT meet AC1-AC7 (which require **setup instructions** about Company Structure, payroll readiness, etc.).

---

**Test conducted by:** 428HR Engineer
**Total test duration:** ~25 minutes
**Result:** Issue #464 must be reopened and the fix implemented.
