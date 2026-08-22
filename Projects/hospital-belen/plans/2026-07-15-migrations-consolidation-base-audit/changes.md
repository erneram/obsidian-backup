# Changes — stage-2 effort=high (auditoría + consolidación) + P3 gates fix

## P2.1 — Fix slug bug `default` → `hospital-belen`

**`hospital-belen-api/src/infrastructure/database/repositories/tenant_repository.rs`**
- `WHERE slug = 'default'` → `WHERE slug = 'hospital-belen'`; error message updated

**`hospital-belen-api/migrations/002_seed.sql`** (ex-002_seed_rbac.sql)
- Tenant INSERT: `'hospital-belen'`, `ON CONFLICT (id)`
- All `WHERE slug/t.slug = 'default'` → `'hospital-belen'`

---

## P2 — Consolidación 11 → 3 migraciones

**`hospital-belen-api/migrations/001_schema.sql`**
- Header: squash 001–032; `stmt_concept` + 10 valores ES (fold 028); `account_statement_items` + columns + 2 indexes (fold 027); `receipts` + `concepto/detalle` (fold 027)

**`hospital-belen-api/migrations/002_seed.sql`** (renombrado de 002_seed_rbac.sql)
- Roles: slugs finales `tenant_admin`/`secretary` (fold 026)
- menu_item_roles `tenant_admin`: `NOT IN ('roles-list','endpoints','admin-tenants')` (fold 031)

**`hospital-belen-api/migrations/003_seed_reference.sql`** ← NEW
- Merge: 003_seed_surgery_packages + 004_seed_who_growth

**Eliminados:** 003, 004, 026–032 incrementales

---

## P3-V1 fix — migrations.bak poblado

**`hospital-belen-api/migrations.bak/`**
- Repoblado con 9 archivos de `git show HEAD:migrations/` (001–004, 026–030)
- V1 parity gate (`verify_migrations.sh`) ahora tiene baseline válido

---

## P3-V2

**`scripts/verify_seed_invariants.sh`** ← NEW
- 10 invariantes SQL tras aplicar 3 consolidados

---

## P3-V3 fix — HTTP real endpoint, no inline SQL

**`hospital-belen-api/tests/tenant_provisioning.rs`** (reescrito)
- POST `/api/tenants` como `platform_super_admin` (superadmin@system.local/111111)
- Verifica counts en DB: roles/role_permissions/menu_item_roles == template `hospital-belen`
- Rollback via `DELETE FROM tenants WHERE id = ...`
- Falla ANTES del fix (slug `hospital-belen` no encontrado → counts=0)
- Verde POST-fix

---

## Cleanup / P1

**`hospital-belen-web`**: eliminados ShadcnExamplesPage.vue + ruta
**`hospital-belen-api`**: 2 `#[allow(dead_code)]` innecesarios removidos
**`CLAUDE.md`**: sección "Cuándo usar base.rs" con tabla A–E
**`tenant_repository.rs`**: doc block ponytail con razón del raw

---

## Para el Tester

- `scripts/verify_migrations.sh` — gate V1 (requiere Docker, ~2 min)
- `scripts/verify_seed_invariants.sh` — gate V2 (requiere Docker)
- `cargo test --test tenant_provisioning` — gate V3 (requiere TEST_DATABASE_URL + API corriendo)
- Boot limpio: `docker compose down -v && docker compose up` para validar los 3 archivos
- NO borrar `migrations.bak/` hasta que V1 parity pase
