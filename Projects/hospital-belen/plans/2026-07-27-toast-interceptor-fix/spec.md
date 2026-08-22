# Spec — Fix reviewer issues (Platform Admin toast/interceptor)

**Project:** hospital-belen · **effort:** low · **stage-1 (fix)**
Continuation of the toast/interceptor work. Two reviewer issues only.

## Issue 1 — `removeOverride` toasts success without a success check

**File:** `hospital-belen-web/kairosaid/src/modules/platform/pages/PlatformTenantDetailPage.vue`
**Function:** `removeOverride` (currently lines 627–631)

`platformPlanService.deleteOverride(...)` returns `Promise<boolean>`
(`platformPlan.service.ts:213`). The function ignores it and always mutates state +
toasts success. Guard on the return, matching `saveOverride` (line 607, which guards
`if (result.success)` and relies on the interceptor for error feedback).

Change to:
```ts
async function removeOverride(featureId: string) {
  const ok = await platformPlanService.deleteOverride(tenantId, featureId)
  if (!ok) return
  overrides.value = overrides.value.filter(o => o.featureId !== featureId)
  toast.success('Override eliminado.')
}
```
- Only mutate `overrides` and toast on `ok === true`.
- No explicit `toast.error` — the shared interceptor already toasts 500/network
  (same as `saveOverride`). Do not add one.

## Issue 2 — Parent worktree has out-of-scope deletions

Before staging, restore files this feature never touched. In the **parent repo**
(`~/Documents/tony/projects/hospital-belen`), these are deleted and unrelated to the
toast/interceptor work:

```
docs/solutions/architecture-patterns/pagination-backend-frontend-contract.md
docs/solutions/architecture-patterns/uniform-pagination-contract.md
docs/solutions/design-patterns/rbac-role-creation-tenant-admin.md
```

Restore them:
```bash
git -C ~/Documents/tony/projects/hospital-belen checkout -- \
  docs/solutions/architecture-patterns/pagination-backend-frontend-contract.md \
  docs/solutions/architecture-patterns/uniform-pagination-contract.md \
  docs/solutions/design-patterns/rbac-role-creation-tenant-admin.md
```

- Do **not** stage `.DS_Store` (noise).
- `.pipeline/*` churn is pipeline artifacts — leave as-is, don't stage.
- Only the in-scope change lives in the `hospital-belen-web` submodule.

## Branch

`.pipeline/branch.md` → `fenix/platform-toast-interceptor`. The web submodule is
currently on `main` with uncommitted feature changes — move them onto the branch;
do not commit to `main`.

## Out of scope

- No new toasts, no interceptor changes beyond Issue 1's guard.
- No changes to `docs/solutions/*` content (only restore the deletions).

## Verification

- Delete an override while backend fails (500) → no success toast, row stays.
- Delete an override successfully → row removed, "Override eliminado." toast.
- `git status` in parent shows the 3 docs files no longer deleted.
