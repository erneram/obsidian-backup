# Spec — Fix 500 on platform admin user deletion

## Problem
`DELETE /api/platform/tenants/{tenant_id}/users/{user_id}` returns 500 even when
authenticated as a platform admin. Reproduced on kairosaid.inksightdev.com.

## Root cause
`UserBmc` deletes via soft-delete (`DeleteMode::SoftSetField { column: "status", value: "INACTIVE" }`),
which runs `UPDATE users SET status = $1`. `users.status` is Postgres enum `user_status`
(`migrations/001_schema.sql:375`, type at `:37`). The bind param `$1` is typed `text`, and
Postgres has **no implicit `text → user_status` cast**, so the query fails →
`DomainError` → 500.

Every other enum write in the codebase casts explicitly (e.g.
`endpoint_repository.rs:143` `.cast_as(Alias::new("http_method"))`;
`user_repository.rs:175` `'ACTIVE'::user_status`). The soft-delete path omits the cast.
Isolated bug: `UserBmc` is the **only** `SoftSetField` user (grep confirms).

## Fix
Add an optional enum cast to the soft-delete-set-field path.

### 1. `src/infrastructure/database/base.rs`
- Extend the `DeleteMode::SoftSetField` variant with a cast type:
  ```rust
  SoftSetField {
      column: &'static str,
      value: &'static str,
      cast: Option<&'static str>,   // Postgres type to cast the value to, e.g. "user_status"
  },
  ```
- In `_soft_delete_set_field`, replace the plain bind:
  ```rust
  .value(Alias::new(column), value)
  ```
  with a cast-aware bind. Signature gains `cast: Option<&'static str>`:
  ```rust
  let val: SimpleExpr = match cast {
      Some(ty) => Expr::val(value).cast_as(Alias::new(ty)),
      None => value.into(),
  };
  query.table(MC::table_ref())
       .value(Alias::new(column), val)
       ...
  ```
- Update the `match MC::DELETE_MODE` arm (base.rs:364) to destructure and forward `cast`:
  ```rust
  DeleteMode::SoftSetField { column, value, cast } =>
      _soft_delete_set_field::<MC>(ctx, db, id, column, value, cast).await,
  ```
- Ensure `SimpleExpr` / `Expr` / `Alias` are imported in base.rs (Alias already used).

### 2. `src/infrastructure/database/repositories/user_repository.rs:37`
```rust
const DELETE_MODE: DeleteMode = DeleteMode::SoftSetField {
    column: "status",
    value: "INACTIVE",
    cast: Some("user_status"),
};
```

## Interfaces / signatures
- `_soft_delete_set_field<MC: DbBmc>(ctx, db, id, column: &'static str, value: &'static str, cast: Option<&'static str>) -> Result<()>`

## Edge cases
- `cast: None` must keep current behavior (plain text/varchar columns) — no regression for
  any future non-enum SoftSetField user.
- Deleting an already-INACTIVE user still succeeds (idempotent `UPDATE`, `rows_affected == 1`);
  a non-existent id still yields `rows_affected == 0 → EntityNotFound` (404). Preserve both.
- Tenant scoping (`IS_TENANT_SCOPED`) and `HAS_TIMESTAMPS` branches unchanged.

## Patterns to follow
- Enum cast: `endpoint_repository.rs:143` (`SimpleExpr::from(x).cast_as(Alias::new("http_method"))`).

## Verification
- `cargo build` (workspace compiles with new variant field — all match arms updated).
- Manual/curl: `DELETE /api/platform/tenants/{tenant}/users/{user}` as platform admin → `204`,
  and `users.status` flips to `INACTIVE`.

---

# Part 2 — "Activate tenant" button in danger zone (frontend)

## Goal
In the tenant detail danger zone, add an **"Activate tenant"** button that appears only when
the tenant is currently deactivated (`!tenant.isActive`), so an admin can re-enable a tenant.
Currently the danger zone only offers "Deactivate tenant" (disabled once already inactive) —
there is no way back through the UI.

## Files to modify (repo: `hospital-belen-web/kairosaid`)
- `src/modules/platform/pages/PlatformTenantDetailPage.vue`
- `src/locales/es.json`

## Changes
### `PlatformTenantDetailPage.vue`
1. **Template — danger zone** (`~lines 162-169`). Add an activate button next to the existing
   deactivate button. The deactivate button already carries `:disabled="!tenant.isActive"`, so
   show the activate one conditionally when inactive. Wrap both in a `flex gap-3` row:
   ```html
   <div class="flex gap-3">
     <Button v-if="tenant.isActive" variant="destructive" :loading="deactivating"
             :disabled="!tenant.isActive" @click="handleDeactivate">
       {{ $t('platform.tenants.deactivate') }}
     </Button>
     <Button v-else variant="default" :loading="activating" @click="handleActivate">
       {{ $t('platform.tenants.activate') }}
     </Button>
   </div>
   ```
   (Only one is visible at a time — deactivate when active, activate when inactive.)
2. **Script — state ref** (near `const deactivating = ref(false)`, ~line 515):
   ```ts
   const activating = ref(false)
   ```
3. **Script — handler** (mirror `handleDeactivate`, ~line 796). Calls the same service `update`
   with `isActive: true`:
   ```ts
   async function handleActivate() {
     if (!tenant.value) return
     activating.value = true
     try {
       const result = await platformTenantService.update(tenantId, { isActive: true })
       if (result.success) {
         tenant.value = result.tenant
         form.value.isActive = true
         toast.success('Tenant activado.')
         saveSuccess.value = true
         setTimeout(() => (saveSuccess.value = false), 3000)
       } else {
         saveError.value = result.error
       }
     } finally {
       activating.value = false
     }
   }
   ```

### `src/locales/es.json` (~line 744, next to `"deactivate"`)
```json
"activate": "Activar tenant",
```

## Interfaces / patterns to follow
- Reuse `platformTenantService.update(tenantId, { isActive })` — same call `handleDeactivate` uses.
  No new endpoint/service method.
- `Button` from `@inksightdev/ui`. Available variants: `default | destructive | outline | secondary
  | ghost | link` (no `success`). Confirmed in `inksight-web-components/packages/ui-brand/src/components/Button.vue:44-49`.

## Edge cases
- Button visibility is driven off `tenant.isActive` (the persisted value), not `form.isActive`
  (the unsaved toggle at line 127) — keep them independent so an unsaved form toggle doesn't
  swap the danger-zone button.
- After activate succeeds, `tenant.value` is replaced with the server response and `form.isActive`
  synced to `true`, so the button flips back to "Deactivate" without a reload.
- Guard against double-submit via the `activating` loading state (mirrors `deactivating`).

## Diseño visual
- The danger-zone container stays as-is (`border-destructive/30`, heading `text-destructive`).
- Deactivate = `variant="destructive"` (unchanged). Activate is a **recovery/positive** action →
  `variant="default"` (primary fill), which reads as the affirmative counterpart against the red
  frame. Do **not** use `destructive` for activate.
- Same `Button` size/spacing as the existing button; both sit in one `flex gap-3` row so the
  danger zone keeps a single action line whether the tenant is active or inactive.
- Loading spinner via the existing `:loading` prop — no custom spinner.
