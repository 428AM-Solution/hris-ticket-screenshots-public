# 🐛 [BUG] Super Admin cannot create Cutoff Periods — silent failure with no error message

**Severity:** 🔴 **High** — Blocks payroll processing for ALL tenants (cutoff periods are prerequisites for payroll generation)

**Component:** `app/controllers/CutoffPeriodController.php` — `store()` method

**Affects:** All users, including `super_admin` (admin@428am.com)

**Discovered:** E2E test on 2026-08-23 (manual testing of payroll generation workflow)

**Status:** Open

---

## Summary

The "Create Cutoff Period" feature on `/cutoff-periods/create` **silently fails** for every user, including `super_admin`. The form accepts input, the POST returns HTTP 302 redirecting to `/cutoff-periods`, **no record is created in the database, and no error message is shown to the user**.

The same form was working before recent tenant-scope changes (PR #413 / PR #414 merged on 2026-08-22 per `git log`). This appears to be a **regression**.

---

## Steps to Reproduce

1. Login as **any** user (verified both `demo@gmail.com` and `admin@428am.com` / `00000000`):
   ```
   https://hris.428am.com/login
   ```

2. Navigate to:
   ```
   https://hris.428am.com/cutoff-periods/create
   ```

3. Fill the form:
   - **Company:** 428AM (id=1, status=Active, verified)
   - **Period Start:** 2029-01-01
   - **Period End:** 2029-01-15
   - **Cutoff Name:** any unique string (e.g., "E2E_TEST_2029_01")
   - **Pay Period Type:** First Period

4. Click **Create Cut-off Period**

5. Observe:
   - **URL changes** to `https://hris.428am.com/cutoff-periods`
   - **Table is empty** — no record created
   - **No flash message** — user has no feedback
   - **Cutoff Periods list shows "Ready to plan a pay cycle?" empty state**

---

## Expected vs Actual

| | Expected | Actual |
|---|---|---|
| HTTP | `302 Found` to `/cutoff-periods/{uuid}` (show page) | `302 Found` to `/cutoff-periods` (list) |
| Database | New row in `M_CutoffPeriod` | No row inserted |
| UI feedback | Success flash: "Cutoff period created successfully!" | No message at all |
| Available to use | Yes — visible in `/cutoff-periods` list | No — flow effectively dead |

---

## Evidence (Reproduction Artifacts)

### Test setup
- **Method:** Raw HTTP via `python3` + `requests.Session` (bypasses browser caching, JS, etc.)
- **User agent:** `python-requests/2.x`
- **Server response inspection:** Full headers + body captured for every request
- **Session verification:** Dashboard GET returns `200 OK` with "Super Admin" string before attempting POST

### Live test log (admin@428am.com, super_admin)
```
$ python3 e2e_step9.py
Logged in: https://hris.428am.com/dashboard

POST /cutoff-periods/store
  Status: 302 Found
  Location: https://hris.428am.com/cutoff-periods
  Body: (empty — 0 bytes)
  Set-Cookie: (no session regeneration)

GET https://hris.428am.com/cutoff-periods
  Status: 200
  Empty state: "Ready to plan a pay cycle? Pick a company, year, and status above..."

# Verify record was NOT created:
$ SELECT COUNT(*) FROM M_CutoffPeriod WHERE CutoffName LIKE 'E2E_%';
0
```

### Multi-company test (all company IDs)
| Company ID | Name | Result |
|---|---|---|
| 1 | 428AM (Active, 8 employees) | 302 → /cutoff-periods, no record |
| 105 | Trial - 428 | same |
| 39 | 428AM Tech Solution | same |
| 104 | ACME PWEDE NA | same |

**No company works** — not a tenant-scope data issue, **affects every company**.

### CSRF + session verified intact
- Direct curl with `Cookie: HRIS_SESSION=...` and matching CSRF → same 302 result
- Logged-in user can load `/cutoff-periods/create` (HTTP 200)
- Logged-in user can load `/dashboard` (HTTP 200 with "Super Admin" string)
- Logged-in user can create a Department (`POST /departments` returns 302 → show page with UUID) ← **this works**
- Logged-in user can create an Employee (`POST /employees` returns 200) ← **this fails too but with different error**
- Logged-in user can NOT create a Cutoff Period (`POST /cutoff-periods/store` returns 302 → list) ← **the bug**

---

## Suspected Root Cause (in `app/controllers/CutoffPeriodController.php:store()`)

### The redirect location is the smoking gun

The `store()` method has these redirect locations depending on outcome:

| Outcome | Redirect location |
|---|---|
| Success | `cutoff-periods/{$uuid}` (show page) |
| Tenant-scope check fails | `cutoff-periods` (list page) ← **← we observe this** |
| Duplicate name validation | re-renders create form (HTTP 200) |
| Overlap validation | re-renders create form (HTTP 200) |
| Other validation | re-renders create form (HTTP 200) |
| DB exception | re-renders create form (HTTP 200) |

**Only the tenant-scope failure path** redirects to `/cutoff-periods` (the list). That matches what we observe.

### The tenant-scope logic (line 224-230)

```php
$companyCheck = $this->ensureCompanyInScope((int) ($data['company_id'] ?? 0), $scope);
if (!$companyCheck) {
    $this->setFlash('error', 'You do not have access to the selected company.', 'error');
    $this->redirect('cutoff-periods');
    return;
}
```

### The ensureCompanyInScope helper (line 50-68)

```php
private function ensureCompanyInScope(int $companyId, array $scope): ?array
{
    $company = $this->companyRepository->findById($companyId);
    if (!$company) {
        return null;          // ← (A) Company not found / inactive
    }
    if ($scope['is_super_admin']) {
        return $company;      // ← (B) Super admin bypass
    }
    if ($scope['tenant_id'] > 0 && (int) ($company['TenantID'] ?? 0) === $scope['tenant_id']) {
        return $company;      // ← (C) Tenant match
    }
    if ($scope['tenant_id'] <= 0 && $scope['company_id'] > 0
        && (int) ($company['CompanyID'] ?? 0) === $scope['company_id']) {
        return $company;      // ← (D) Company match
    }
    return null;              // ← (E) No match
}
```

### Most likely cause: branch (A) or branch (B) misbehaving

- **Branch (A)** — `findById(1)` returns null despite company 1 being Active and visible everywhere. Possible causes:
  - DB connection issue between controller's repo and view's repo
  - PDO prepared statement binding issue (int param binding)
  - Schema cache or stale metadata
  - A NEW filter was added (e.g., ReferenceTableStatusID != 1)
  - Cross-tenant filter was added by mistake (e.g., `WHERE TenantID = $scope['tenant_id']` even for super_admin)

- **Branch (B)** — `$scope['is_super_admin']` is false despite session role being `'super_admin'`. Possible causes:
  - `currentUser['role']` is unset/null when `resolveTenantScope()` is called
  - `reconcileRoleFromDb()` runs too late (after `resolveTenantScope`)
  - Session role is `super_admin` (string) but DB has different casing
  - Tenant scope is computed from a DIFFERENT user than the logged-in one

### Additional bug found in code review

**Secondary bug in line 244-247 of `store()`:** A SECOND `ensureCompanyInScope` call exists with an **undefined variable `$cutoffPeriodUuid`**:

```php
if (empty($errors['company_id'])) {
    $companyCheck = $this->ensureCompanyInScope((int) ($data['company_id'] ?? 0), $scope);
    if (!$companyCheck) {
        $this->setFlash('error', 'You do not have access to the selected company.', 'error');
        $this->redirect("cutoff-periods/{$cutoffPeriodUuid}");  // ← $cutoffPeriodUuid NOT defined here!
        return;
    }
}
```

`$cutoffPeriodUuid` is defined inside the `try` block on line 240, AFTER this guard. If the second check fails, the redirect URL would be `cutoff-periods/` (with empty UUID, which would 404). However, since this branch is only hit when `company_id` has NO error but the company is out of scope, and the first check at line 224 already covers super_admin bypass, this code is effectively dead code in normal flow.

**Action:** Should be cleaned up regardless.

---

## Suggested Next Steps (for engineering team)

1. **Check server error logs** for any PHP warnings/exceptions during POST `/cutoff-periods/store` from super_admin user. The user-friendliness of the failure is so poor that we have no client-side signal beyond the redirect location.

2. **Add temporary logging** at line 225 in `store()`:
   ```php
   error_log("[CUTOFF_TRACE] scope=" . json_encode($scope) . " company_id=" . $data['company_id']);
   error_log("[CUTOFF_TRACE] companyCheck=" . json_encode($companyCheck));
   ```

3. **Verify branch (A) by logging** what `findById(1)` returns:
   ```php
   $company = $this->companyRepository->findById($companyId);
   error_log("[CUTOFF_TRACE] findById(1) returned: " . json_encode($company));
   ```

4. **Verify branch (B)** by logging `$scope`:
   ```php
   error_log("[CUTOFF_TRACE] scope is_super_admin: " . ($scope['is_super_admin'] ? 'true' : 'false'));
   ```

5. **Check for recent git changes** to `CutoffPeriodController.php` and `BaseRepository.php` / `CompanyRepository.php` that might have introduced a new filter.

---

## Related Context

### Recent commits that may be related
- `176c3e38` (2026-08-22) — Merge PR #414: hotfix cutoff periods PHP parse error (refs #412)
- `fd6ed467` (2026-08-22) — Merge PR #413: cutoff periods tenant filtering + searchable dropdowns (refs #412)
- `90da0774` (2026-08-09) — feat(security): scope every module by M_Company.TenantID (#254)

### Tests run (artifacts on /tmp)
- `/tmp/e2e_python_admin2.py` — multi-company POST attempt
- `/tmp/raw_trace2.py` — raw http.client trace showing exact response
- `/tmp/e2e_response_body.py` — empty 302 body inspection
- `/tmp/e2e_other_creates.py` — comparison test (departments WORK, shifts FAIL, cutoffs FAIL)
- `/tmp/cutoff_create.html` — saved HTML of the create page

### Tenant-scope comparison
| Endpoint | Tenant guard present? | Super admin can create? |
|---|---|---|
| `POST /departments` | No (departments don't filter by tenant) | ✅ Yes (verified) |
| `POST /employees` | Yes | ❌ No (different failure mode) |
| `POST /shifts` | No (uses shiftGroupId instead) | ✅ Yes for /shifts (but separate /shifts/group endpoint fails) |
| `POST /companies` | No | ✅ Yes |
| `POST /cutoff-periods/store` | **Yes (broken)** | **❌ No (the bug)** |
| `POST /cutoff-periods/generate` | Yes | ❌ No (silent failure, HTTP 200 with no body) |

The cutoff creation bug appears to be a **regression from PR #413 (cutoff periods tenant filtering)** or PR #414 (parse error fix), both merged 2026-08-22.

---

## Suggested Ticket Type

This blocks the **entire payroll workflow** for ALL tenants — there is no workaround. Recommend:

- **Priority:** P0 (Critical) — blocks payroll generation
- **Type:** Bug
- **Estimate:** 2-4 hours (1 hour to identify exact branch failing, 1 hour to fix, 1 hour regression test)
- **Assigned:** Backend engineer (CutoffPeriodController domain)

---

## Workaround

Until this is fixed, **no workaround exists** for creating cutoff periods via the UI. Operations team can insert directly via SQL, but that bypasses the validation/audit trail.

The `POST /cutoff-periods/check-duplicate` endpoint **does work** (returns JSON) — so the controller class is loading and basic DB access works. The bug is specifically in `store()` after the duplicate check.
