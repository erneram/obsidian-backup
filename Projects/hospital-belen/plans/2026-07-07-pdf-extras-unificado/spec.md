# Spec — EC PDF: extras unificados, bodega, anticipos formato
**project:** hospital-belen  
**dispatch:** stage-1 (2026-07-07, effort=medium)

---

## Archivos a modificar

| Archivo | Items |
|---------|-------|
| `hospital-belen-api/src/infrastructure/pdf/mod.rs` | 1, 2, 3 |
| `hospital-belen-api/src/web/handlers/admission.rs` | 1, 2, 3 |
| `hospital-belen-api/src/domain/admission/mod.rs` | 2 |
| `hospital-belen-api/src/domain/admission_extra/mod.rs` | 2 |
| `hospital-belen-api/src/domain/receipt/mod.rs` | 3 |
| `hospital-belen-api/src/infrastructure/database/repositories/admission_repository.rs` | 2 |
| `hospital-belen-api/src/infrastructure/database/repositories/admission_extra_repository.rs` | 2 |

---

## Item 1 — Extras conceptuales → único recuadro "Extras"

### Diagnóstico

Actualmente los items de `account_statement_items` se reparten en 4 secciones:
- HOSPITALIZACIÓN/EXTRAS → HOSPITAL, HOSPITALIZACION + admission_extras
- OTROS GASTOS → OTHER, ROOM, ADVANCE, OTROS, MEDICAMENTOS, MATERIAL_QUIRURGICO
- LABORATORIOS → LABORATORY, LABORATORIO, RAYOS_X, ULTRASONIDO
- HONORARIOS MÉDICOS → FEES, HONORARIOS_MEDICOS, QUIROFANO, ANESTESIA

Nueva estructura:
- **PAQUETE** → concept=PACKAGE (sin cambio)
- **EXTRAS** → TODOS los conceptos excepto PACKAGE y OTHER (ver Item 2)
- **EXTRAS DE INVENTARIO** → concept=OTHER + admission_extras (ver Item 2)

### Cambios en `pdf/mod.rs`

**1. Nueva constante para sección conceptual (3 cols):**
```rust
// EC conceptual section: tipo | descripción | monto (sin subtotal)
const EC_SIMPLE_COLS: [usize; 3] = [3, 8, 2];
```

**2. Nueva función `ec_section_conceptual`** (3 columnas, sin subtotal_label):
```rust
fn ec_section_conceptual(
    doc: &mut Document,
    title: &str,
    rows: &[(String, String, f64)],
) -> Result<(), String> {
    if rows.is_empty() { return Ok(()); }
    doc.push(Break::new(0.5));
    doc.push(Paragraph::new(genpdf::style::StyledString::new(
        title,
        Style::new().bold().with_font_size(9).with_color(TEAL),
    )));
    doc.push(Break::new(0.2));

    let td = Style::new().with_font_size(9).with_color(INK);
    let td_muted = Style::new().with_font_size(9).with_color(MUTED);

    let mut table = TableLayout::new(EC_SIMPLE_COLS.to_vec());
    table.set_cell_decorator(FrameCellDecorator::new(false, true, false));

    let mut subtotal = 0.0;
    for (tipo, desc, amount) in rows {
        subtotal += *amount;
        let mut row = table.row();
        row.push_element(cp(Paragraph::new(genpdf::style::StyledString::new(tipo.as_str(), td)).aligned(Alignment::Left)));
        row.push_element(cp(Paragraph::new(genpdf::style::StyledString::new(desc.as_str(), td_muted)).aligned(Alignment::Left)));
        row.push_element(cp(Paragraph::new(genpdf::style::StyledString::new(format!("Q {:.2}", amount), td)).aligned(Alignment::Right)));
        row.push().map_err(|e| e.to_string())?;
    }
    // Subtotal row
    {
        let mut row = table.row();
        row.push_element(cp(Paragraph::new(genpdf::style::StyledString::new("", Style::new()))));
        row.push_element(cp(Paragraph::new(genpdf::style::StyledString::new(
            "Subtotal",
            Style::new().bold().with_font_size(8).with_color(MUTED),
        )).aligned(Alignment::Right)));
        row.push_element(cp(Paragraph::new(genpdf::style::StyledString::new(
            format!("Q {:.2}", subtotal),
            Style::new().bold().with_font_size(9).with_color(INK),
        )).aligned(Alignment::Right)));
        row.push().map_err(|e| e.to_string())?;
    }
    doc.push(table);
    Ok(())
}
```

**3. Reemplazar las 4 secciones actuales por 2** en `generate_estado_de_cuenta_pdf`:

```diff
-    // PAQUETE
-    ec_section(&mut doc, "PAQUETE", &collect(&["PACKAGE"]))?;
-
-    // HOSPITALIZACIÓN / EXTRAS = HOSPITAL items + nursing extras
-    let mut hosp = collect(&["HOSPITAL", "HOSPITALIZACION"]);
-    for e in &data.extras { ... }
-    ec_section(&mut doc, "HOSPITALIZACIÓN / EXTRAS", &hosp)?;
-
-    // OTROS GASTOS
-    ec_section(&mut doc, "OTROS GASTOS", &collect(&["OTHER", ...]))?;
-
-    // LABORATORIOS
-    ec_section(&mut doc, "LABORATORIOS", &collect(&[...]))?;
-
-    // HONORARIOS MÉDICOS
-    ec_section(&mut doc, "HONORARIOS MÉDICOS", &collect(&[...]))?;
+    // ── PAQUETE ────────────────────────────────────────────────────────────────
+    ec_section(&mut doc, "PAQUETE", &collect(&["PACKAGE"]))?;
+
+    // ── EXTRAS (todos los conceptos directos, sin bodega) ──────────────────────
+    const CONCEPTUAL_TYPES: &[&str] = &[
+        "HOSPITAL", "HOSPITALIZACION",
+        "LABORATORY", "LABORATORIO",
+        "FEES", "HONORARIOS_MEDICOS", "QUIROFANO", "ANESTESIA",
+        "RAYOS_X", "ULTRASONIDO",
+        "ROOM", "ADVANCE",
+        "MEDICAMENTOS", "MATERIAL_QUIRURGICO",
+        "OTROS",
+    ];
+    let conceptual_rows: Vec<(String, String, f64)> = data.items
+        .iter()
+        .filter(|i| CONCEPTUAL_TYPES.contains(&i.concept_type.as_str()))
+        .map(|i| (concept_label(&i.concept_type).to_string(), i.description.clone(), i.amount))
+        .collect();
+    ec_section_conceptual(&mut doc, "EXTRAS", &conceptual_rows)?;
+
+    // ── EXTRAS DE INVENTARIO (bodega) ──────────────────────────────────────────
+    // (ver Item 2)
```

---

## Item 2 — Extras de Inventario: unificar + columna Bodega

### Nueva estructura

Sección única "EXTRAS DE INVENTARIO" con 4 columnas usando `EC_COLS`:
`Bodega | Nombre | Unidad×Precio | Monto`

Fuentes de datos:
- `account_statement_items` con concept=`OTHER` (de `data.items`)
- `admission_extras` (de `data.extras`)

Ambos necesitan `storage_name: Option<String>`.

### Análisis de NULLs

| Campo | Fuente | ¿Puede ser NULL? |
|-------|--------|-----------------|
| `quantity` | account_statement_item | `Option<f64>` — puede ser NULL (fallback `1.0` en el INSERT) |
| `unit_price` | account_statement_item | `Option<f64>` — puede ser NULL (no hay fallback) |
| `quantity` | admission_extra | `i32` — nunca NULL |
| `unit_price` | admission_extra | `f64` — nunca NULL |
| `storage_id` | account_statement_item | `Option<Uuid>` — puede ser NULL (items no-inventario) |
| `storage_id` | admission_extra | `Option<Uuid>` — puede ser NULL |

Columna "Unidad×Precio": mostrar `"N × Q p.pp"` cuando ambos están disponibles, `"—"` si unit_price es NULL.

### Cambios en `domain/admission/mod.rs`

Agregar `storage_name` a `AccountStatementItem`:
```diff
 pub struct AccountStatementItem {
     pub id: Uuid,
     ...
     pub unit_price: Option<f64>,
+    pub storage_name: Option<String>,
     pub created_at: String,
 }
```

### Cambios en `domain/admission_extra/mod.rs`

Agregar `storage_name` a `AdmissionExtra`:
```diff
 pub struct AdmissionExtra {
     ...
     pub storage_id: Option<Uuid>,
+    pub storage_name: Option<String>,
     pub description: String,
```

### Cambios en `repositories/admission_repository.rs` — query items

Agregar LEFT JOIN con storages en el `SELECT` de items (dentro de `get`, líneas ~192-206):
```diff
-            SELECT
-                id, tenant_id, account_statement_id,
-                concept::text           AS concept,
-                description,
-                original_amount::float8 AS original_amount,
-                paid_amount::float8     AS paid_amount,
-                quantity::float8        AS quantity,
-                unit_price::float8      AS unit_price,
-                created_at::text        AS created_at
-            FROM account_statement_items
-            WHERE account_statement_id = $1 AND tenant_id = $2
+            SELECT
+                asi.id, asi.tenant_id, asi.account_statement_id,
+                asi.concept::text           AS concept,
+                asi.description,
+                asi.original_amount::float8 AS original_amount,
+                asi.paid_amount::float8     AS paid_amount,
+                asi.quantity::float8        AS quantity,
+                asi.unit_price::float8      AS unit_price,
+                s.name                      AS storage_name,
+                asi.created_at::text        AS created_at
+            FROM account_statement_items asi
+            LEFT JOIN storages s ON s.id = asi.storage_id
+            WHERE asi.account_statement_id = $1 AND asi.tenant_id = $2
```

### Cambios en `repositories/admission_extra_repository.rs` — list query

Agregar LEFT JOIN con storages (dentro de `list`, líneas ~38-56):
```diff
-            SELECT
-                id, tenant_id, admission_id, inventory_item_id, storage_id,
-                description, quantity,
-                unit_price::float8  AS unit_price,
-                total::float8       AS total,
-                statement_item_id, created_by,
-                created_at::text    AS created_at
-            FROM admission_extras
-            WHERE admission_id = $1 AND tenant_id = $2
+            SELECT
+                ae.id, ae.tenant_id, ae.admission_id, ae.inventory_item_id, ae.storage_id,
+                ae.description, ae.quantity,
+                ae.unit_price::float8   AS unit_price,
+                ae.total::float8        AS total,
+                ae.statement_item_id, ae.created_by,
+                ae.created_at::text     AS created_at,
+                s.name                  AS storage_name
+            FROM admission_extras ae
+            LEFT JOIN storages s ON s.id = ae.storage_id
+            WHERE ae.admission_id = $1 AND ae.tenant_id = $2
```

### Cambios en `pdf/mod.rs` — `ExtraPdfLine`

```diff
 pub struct ExtraPdfLine {
+    pub storage_name: Option<String>,
     pub description: String,
     pub quantity: i32,
     pub unit_price: f64,
     pub total: f64,
 }
```

### Cambios en `handlers/admission.rs` — mapear storage_name

```diff
 let extra_lines: Vec<ExtraPdfLine> = extras
     .iter()
     .map(|e| ExtraPdfLine {
+        storage_name: e.storage_name.clone(),
         description: e.description.clone(),
         quantity: e.quantity,
         unit_price: e.unit_price,
         total: e.total,
     })
     .collect();
```

### Cambios en `pdf/mod.rs` — nueva sección EXTRAS DE INVENTARIO

Reemplazar el bloque `HOSPITALIZACIÓN/EXTRAS` + `OTROS GASTOS` por una sola sección que combina items OTHER + admission_extras:

```rust
// ── EXTRAS DE INVENTARIO ──────────────────────────────────────────────────
let mut inv_rows: Vec<(String, String, Option<String>, f64)> = Vec::new();

// Items de inventario (concept=OTHER)
for i in data.items.iter().filter(|i| i.concept_type == "OTHER") {
    let bodega = i.storage_name.clone().unwrap_or_else(|| "—".to_string());
    let sub = match (i.quantity, i.unit_price) {
        (Some(q), Some(p)) => Some(format!("{:.0} × Q{:.2}", q, p)),
        _ => None,
    };
    inv_rows.push((bodega, i.description.clone(), sub, i.amount));
}

// Extras de enfermería (admission_extras)
for e in &data.extras {
    let bodega = e.storage_name.clone().unwrap_or_else(|| "—".to_string());
    let sub = Some(format!("{} × Q{:.2}", e.quantity, e.unit_price));
    inv_rows.push((bodega, e.description.clone(), sub, e.total));
}

ec_section(&mut doc, "EXTRAS DE INVENTARIO", &inv_rows)?;
```

Columnas usando EC_COLS `[2, 7, 3, 2]`:
- col1 (2): Bodega
- col2 (7): Nombre del item
- col3 (3): Unidad×Precio
- col4 (2): Monto

---

## Item 3 — Anticipos: formato DD/MM/YYYY + número de recibo + concepto

### Diagnóstico

Actualmente: movements filtrados por `PAYMENT` → solo tienen `date, amount, notes`. Sin número de recibo ni concepto.

El `receipt_repo` ya está en `AppState` y tiene `list(admission_id_filter: Option<Uuid>)`. Se puede usar para traer los talonarios de la admisión.

### Cambios en `domain/receipt/mod.rs`

El `Receipt` struct ya tiene `receipt_number`, `concepto`, `detalle`, `total_amount`. No requiere cambios al dominio.

### Cambios en `pdf/mod.rs`

**1. Nueva función de fecha DD/MM/YYYY:**
```rust
fn format_date_ddmmyyyy(iso: &str) -> String {
    let d = iso.split('T').next().unwrap_or(iso);
    let d = d.split(' ').next().unwrap_or(d);
    let p: Vec<&str> = d.split('-').collect();
    if p.len() < 3 { return iso.to_string(); }
    format!("{}/{}/{}", p[2], p[1], p[0])
}
```

**2. Modificar `PaymentPdfLine`:**
```diff
 pub struct PaymentPdfLine {
     pub date: String,
+    pub receipt_number: String,
+    pub concepto: Option<String>,
     pub amount: f64,
-    pub notes: Option<String>,
 }
```

**3. Anticipos section en `generate_estado_de_cuenta_pdf`:**

Cambiar el mapeo de `pay_rows` para usar receipt_number y concepto:
```diff
-    if !data.payments.is_empty() {
-        let pay_rows: Vec<(String, String, Option<String>, f64)> = data
-            .payments
-            .iter()
-            .map(|p| (
-                format_date_spanish(&p.date),
-                p.notes.clone().unwrap_or_else(|| "Anticipo".to_string()),
-                None,
-                p.amount,
-            ))
-            .collect();
-        ec_section(&mut doc, "ANTICIPOS", &pay_rows)?;
-    }
+    if !data.payments.is_empty() {
+        let pay_rows: Vec<(String, String, Option<String>, f64)> = data
+            .payments
+            .iter()
+            .map(|p| (
+                format!("N° {}", p.receipt_number),
+                p.concepto.clone().unwrap_or_else(|| "Anticipo".to_string()),
+                Some(format_date_ddmmyyyy(&p.date)),
+                p.amount,
+            ))
+            .collect();
+        ec_section(&mut doc, "ANTICIPOS", &pay_rows)?;
+    }
```

### Cambios en `handlers/admission.rs`

**1. Importar Receipt:**
```diff
+use crate::domain::receipt::Receipt;
```

**2. Cambiar pagos a usar receipts** (en `download_statement_pdf`):
```diff
-    let movements = state.admission_repo.list_movements(tenant_id, id).await?;
+    let (receipts, _) = state.receipt_repo.list(&ctx, tenant_id, 1, 200, None, Some("ACTIVE".to_string()), Some(id)).await?;
```

**3. Mapear receipts a PaymentPdfLine:**
```diff
-    let payments: Vec<PaymentPdfLine> = movements
-        .iter()
-        .filter(|m| m.movement_type == "PAYMENT")
-        .map(|m| PaymentPdfLine {
-            date: m.created_at.clone(),
-            amount: m.amount,
-            notes: m.notes.clone(),
-        })
-        .collect();
+    let payments: Vec<PaymentPdfLine> = receipts
+        .iter()
+        .map(|r| PaymentPdfLine {
+            date: r.created_at.clone(),
+            receipt_number: r.receipt_number.clone(),
+            concepto: r.concepto.clone(),
+            amount: r.total_amount,
+        })
+        .collect();
```

**Nota**: `list_movements` puede quedar si se usa para otros fines (el handler ya no lo llama para Anticipos).

---

## Edge cases

- **Item 1**: Si un item tiene concept=OTROS (español) y amount sin qty/price, aparecerá en EXTRAS sin subtotal label. Correcto.
- **Item 2**: `account_statement_items` con concept=OTHER creados via `add_item` (sin storage_id) tendrán `storage_name = NULL` → mostrar "—". OK.
- **Item 2**: El JOIN de `admission_repository.get` afecta a TODOS los items, no solo los OTHER. Para PACKAGE/conceptuales, `storage_name` será NULL. No importa porque en la sección EXTRAS no se usa `storage_name`.
- **Item 3**: Solo se muestran receipts ACTIVE (no anulados). El `paid_amount` del account_statement podría no coincidir si hay receipts anulados — pero ese dato viene de los movimientos, no de los receipts aquí. Sin impacto en totales.
- **Item 3**: Si no hay receipts (`payments.is_empty()`), la sección ANTICIPOS no se renderiza. Correcto.
