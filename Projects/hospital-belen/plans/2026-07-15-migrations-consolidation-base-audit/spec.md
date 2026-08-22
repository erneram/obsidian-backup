# Spec — Auditoría base.rs + consolidación de migraciones + verificación de seeds

**Proyecto:** hospital-belen · **Stage:** 1 (planner) · **Effort:** high
**Fecha:** 2026-07-15 · **Tipo:** auditoría + refactor dev-only · **Estado:** listo para coder.

**Contexto cerrado por el humano:** todos los datos son seededados, **no hay prod**, el
**squash es seguro**. → Se puede recrear la DB libremente; no se requiere migración
forward preservando datos.

**DISPATCH:**
- P1 base.rs: documentar regla (CRUD single-table → base.rs; joins/agg → raw) + refactor cat. E.
- P2 migraciones: consolidar 11→3, fix bug `default→hospital-belen` (029/032/clone), quitar 030.
- P3 seeds: plan V1/V2/V3 + estrategia de squash.

> Capa: los handlers son finos (extraen `Ctx`/`Query`/`Json`, delegan a repo/service). El
> "patrón base.rs" vive en la capa de **repositorios** (`infrastructure/database/base.rs`
> = CRUD genérico single-table vía modql + sea-query). La auditoría se hace a ese nivel.

---

## Parte 1 — Regla base.rs vs raw (documentar) + refactor cat. E

### Qué ofrece `base.rs`
`create/get/list/count/update/delete` genéricos para entidades **single-table**, con
config vía `DbBmc` (`TABLE`, `HAS_TIMESTAMPS`, `DELETE_MODE`, `LIST_FALLBACK_ORDER`,
`IS_TENANT_SCOPED`). Requiere que la entidad **derive modql `Fields`** (`HasSeaFields`).
Solo 9 entidades lo derivan hoy: `role, permission, endpoint_group, endpoint,
medical_result, user, audit_log, menu, growth_point`.

### Clasificación de los 24 repos
| Cat | Repos | Acción |
|---|---|---|
| **A. base limpio** | permission, endpoint_group, menu_item, patient_growth_point, audit_log | dejar |
| **B. mixto** (base + raw puntual) | endpoint, medical_result, role, user, patient | dejar |
| **C. raw justificado** (joins/agg/search) | inventory, admission, package, receipt, doctor, appointment, admission_extra, who_growth | dejar (raw correcto) |
| **D. junction M2M** (base no modela M2M) | user_role, menu_item_role | dejar (raw correcto) |
| **E. raw evitable** (CRUD simple reinventado) | tenant (get/list/find_by_slug), patient_contact, reminder | **refactor** |

### 1A. Documentar la regla — `hospital-belen-api/CLAUDE.md`
Añadir sección **"Cuándo usar base.rs"**:
> CRUD single-table tenant-scoped → `base.rs` (deriva `Fields` + impl `DbBmc`).
> Joins, agregados (SUM/COUNT), búsqueda (ILIKE), upsert masivo o tablas junction (M2M)
> → raw `sqlx` con `#![allow(clippy::disallowed_methods)]` **y comentario del porqué**.

### 1B. Refactor categoría E (solo lecturas/borrado triviales)
- **tenant_repository**: migrar `get/list/find_by_slug/delete` a `base.rs`. **Conservar
  `create` raw** (tiene la lógica de clonado de plantilla). Requiere: entidad `Tenant`
  derive `Fields`; `TenantBmc: DbBmc { TABLE="tenants"; IS_TENANT_SCOPED=false }`
  (los tenants no son tenant-scoped entre sí).
- **patient_contact_repository** y **reminder_repository**: si su CRUD es single-table,
  derivar `Fields` + `DbBmc` y usar base para `get/list/create/update/delete`.
- `// ponytail:` refactor de bajo valor — hacerlo porque se toca el área; no reescribir
  nada de A/B/C/D.

---

## Parte 2 — Consolidación de migraciones 11 → 3 (dev-only)

### Estado actual
`001_schema`(1714) · `002_seed_rbac`(945) · `003_seed_surgery_packages`(3572) ·
`004_seed_who_growth`(1247) · incrementales `026-032`. Hueco 005-025 ya squasheado;
`migrations.bak/` vacío.

### Deudas a corregir DURANTE el fold
1. **BUG `default` vs `hospital-belen`** — `029` renombra el tenant fundacional a
   `hospital-belen`, pero `032_backfill` y `tenant_repository.rs:177` (clonado) buscan
   `slug='default'` → provisioning no-op silencioso. **Canonizar `hospital-belen`** en:
   `002_seed.sql` (crear el tenant ya con ese slug), `tenant_repository.rs` (lookup de
   plantilla), y cualquier `WHERE slug='default'`. Grep de control:
   `grep -rn "'default'" src migrations`.
2. **030 redundante** — re-declara `quantity/unit_price` en `account_statement_items`
   (ya añadidas por `027` como `NOT NULL DEFAULT`), y en nullability distinta. **Eliminar
   030**; dejar la definición final única en `001_schema` (`quantity NUMERIC(10,2) NOT
   NULL DEFAULT 1`, `unit_price NUMERIC(10,2)` nullable).
3. **026 renames post-seed** — sembrar slugs **finales** directamente: `tenant_admin`
   (no `admin`), `secretary` (no `receptionist`).
4. **028 enum incremental** — declarar `stmt_concept` **completo** (EN + ES) en el
   `CREATE TYPE` de `001_schema`; sin `ALTER TYPE ADD VALUE`.
5. **031 exclusión** — **no** asignar `roles-list`/`endpoints`/`admin-tenants` a
   `tenant_admin` en el seed (en vez de asignar y luego borrar).
6. **027 columnas** — foldear a `001_schema`: `receipts.concepto/detalle`,
   `account_statement_items.inventory_item_id/storage_id` + índices.

### Objetivo — 3 archivos con estado final
```
001_schema.sql        DDL final: tablas + columnas + enums completos + índices.
                      Folds: 027, 028 (enum full), 030 (dedup + nullability final).
002_seed.sql          Tenant fundacional slug=hospital-belen + roles con slugs finales
                      (fold 026) + permissions + role_permissions + menu_items +
                      menu_item_roles SIN platform-only para tenant_admin (fold 031).
                      Plantilla de clonado consistente = hospital-belen.
003_seed_reference.sql  surgery packages (ex-003) + WHO growth (ex-004).
```
`032_backfill` **se elimina**: en DB fresca no hay tenants huérfanos; el clonado para
tenants futuros vive en `tenant_repository.create`.

### Mecánica del migrador (custom)
`migrator.rs` descubre `*.sql`, ordena por nombre, trackea por `name`+`checksum`, salta
aplicados. Consolidar cambia los nombres → **recrear la DB** (`docker compose down -v`
+ up). Seguro: todo es seed, no hay prod. Documentar el paso en el PR.

---

## Parte 3 — Verificación de seeds + estrategia de squash

### Estrategia de squash (segura)
1. Rama de trabajo; **no** borrar los archivos históricos hasta que V1 pase.
2. Generar los 3 consolidados en paralelo a los históricos.
3. Correr **V1 parity** (gate). Si diff vacío → reemplazar históricos por los 3 nuevos y
   borrar `migrations.bak/` + `030`.
4. `docker compose down -v && up` para validar boot limpio desde cero.

### V1 — Parity check (consolidado vs histórico)
`scripts/verify_migrations.sh`: levanta dos Postgres efímeros; (a) aplica los 3 nuevos,
(b) aplica la cadena histórica 001…032; compara `pg_dump --schema-only` normalizado
(ordenar, quitar comentarios/`schema_migrations`). **El diff DEBE ser vacío** → prueba
que el squash no alteró el esquema. Es el gate de la Parte 3.

### V2 — Invariantes de seed (asserts SQL tras aplicar consolidados)
- Tenant fundacional con slug `hospital-belen` (no `default`).
- Roles con slugs finales presentes: `platform_super_admin, tenant_admin, doctor, nurse,
  secretary, pharmacist, billing, lab_tech, viewer, pediatrician`.
- **Cada rol operativo tiene ≥1 `menu_item_roles`** (nadie nace sin módulos).
- **`tenant_admin` NO tiene** `roles-list`/`endpoints`/`admin-tenants`.
- Conteo de `permissions` por recurso == esperado.
- Cero `menu_item_roles` huérfanos (FKs válidas).

### V3 — Provisioning de tenant nuevo (integración Rust)
Crear tenant vía `tenant_repository.create` con plantilla → assert que
roles/role_permissions/menu_item_roles del tenant nuevo **igualan en conteo** a los de la
plantilla `hospital-belen`, y que `get_modules` para un usuario de ese tenant devuelve sus
módulos. **Este test falla hoy** por el bug del slug → su verde confirma el fix.

**Ubicación:** V1 = `scripts/verify_migrations.sh`; V2/V3 = tests de integración en
`tests/` contra Postgres de test. Correr en CI antes de mergear.

---

## Orden de ejecución (coder)
1. **P2.1 fix slug** `default→hospital-belen` (código + seed) — arregla el bug, desbloquea V3.
2. **P2 consolidar** a 3 archivos (folds 027/028/030/026/031; eliminar 032).
3. **P3-V1 parity** — GATE: no continuar si el diff no es vacío.
4. **P3-V2/V3** invariantes + provisioning.
5. **Cleanup**: borrar `migrations.bak/`, `030`, `ShadcnExamplesPage.vue` + su ruta;
   auditar `#[allow(dead_code)]` en repos (borrar métodos muertos o quitar el allow).
6. **P1** doc de la regla en CLAUDE.md + refactor cat. E.

---

## Edge cases / riesgos
- **Nullability `quantity`**: definición final `NOT NULL DEFAULT 1`; verificar que el Rust
  que lee `account_statement_items` no asuma `Option`.
- **`stmt_concept` completo**: declarar todos los valores (EN+ES) en el `CREATE TYPE`;
  verificar que ningún código dependa del orden ordinal del enum.
- **Recreación de DB**: paso manual explícito (`down -v`); no auto-dropear en el arranque.
- **Orden de aplicación**: mantener prefijos `001/002/003` para que el sort del migrador
  ejecute schema → seed → reference en ese orden.
- **V1 normalización**: `pg_dump` puede emitir orden no determinista de constraints;
  normalizar (sort) antes de `diff` para evitar falsos positivos.

## SKILL_RECOMENDADA
Ninguna. `pg_dump` + tests de integración sqlx cubren la verificación sin dependencias
nuevas; el resto es refactor de migraciones/repos propios.

## Fuera de alcance
- Migrar cat. C/D a base.rs (sería regresión: perdería joins/agregados/junction).
- Sustituir el migrador custom por `sqlx migrate` (feature aparte; el custom funciona).
- Backfill de tenants huérfanos como migración (innecesario en DB fresca; si se quiere
  para alguna DB dev viva, dejarlo como `scripts/`, no como migración).
