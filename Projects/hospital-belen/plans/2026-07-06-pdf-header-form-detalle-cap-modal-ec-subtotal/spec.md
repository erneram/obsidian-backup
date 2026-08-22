# Spec — 5 fixes: PDF header, form Detalle, amount cap, modal size, EC PDF subtotal
**project:** hospital-belen  
**dispatch:** stage-1 (2026-07-06, effort=medium)

---

## Archivos a modificar

| Archivo | Items |
|---------|-------|
| `hospital-belen-web/kairosaid/src/modules/receipts/components/ReceiptPreview.vue` | 1 |
| `hospital-belen-web/kairosaid/src/modules/receipts/components/ReceiptForm.vue` | 2, 3 |
| `hospital-belen-web/kairosaid/src/modules/receipts/pages/ReceiptCreatePage.vue` | 3 |
| `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue` | 4 |
| `hospital-belen-api/src/infrastructure/pdf/mod.rs` | 5 |
| `hospital-belen-api/src/web/handlers/admission.rs` | 5 |

---

## Item 1 — Quitar header "kairosaid http://localhost:5173/receipts/<id>" del PDF

### Diagnóstico

El texto no viene del backend (genpdf `SimplePageDecorator` no inyecta headers de texto en 0.2.0). Viene del botón **Imprimir** → `window.print()` en `ReceiptPreview.vue`. Chrome añade por defecto:
- Header: `<title de la pestaña>  <URL de la página>` → "kairosaid  http://localhost:5173/receipts/<id>"
- Footer: fecha + número de página

### Fix

**`ReceiptPreview.vue`** — agregar regla `@page` para suprimir el header/footer del navegador:

```diff
 <style>
+@page {
+  margin: 0;
+}
+
 @media print {
   body * {
     visibility: hidden;
   }
   .receipt-paper,
   .receipt-paper * {
     visibility: visible;
   }
   .receipt-paper {
     position: fixed;
     inset: 0;
     max-width: 100%;
+    padding: 10mm;
+    box-sizing: border-box;
   }
 }
 </style>
```

`@page { margin: 0 }` elimina el área de header/footer que Chrome reserva. El `padding: 10mm` compensa para que el contenido no toque los bordes del papel.

---

## Item 2 — Form talonario: eliminar Observaciones, mover Detalle al fondo

### Situación actual

- **Concepto + Detalle** (líneas 36-69): grid de 2 columnas.
- **Notas/Observaciones** (líneas 173-183): `Textarea` con `v-bind="notesAttrs"` al final del form.
- `notes` en el schema Zod: `z.string().optional()` — no es requerido.

### Cambios en `ReceiptForm.vue`

**1. Quitar la sección Notes/Observaciones** (líneas 173-183) — eliminar todo el div:
```diff
-    <!-- Notes -->
-    <div class="space-y-1">
-      <label class="text-sm font-medium text-foreground">{{ $t('receipts.form.notes') }}</label>
-      <Textarea
-        v-bind="notesAttrs"
-        v-model="notes"
-        :rows="3"
-        :placeholder="$t('receipts.form.notesPlaceholder')"
-        class="..."
-      />
-    </div>
```

**2. Extraer Detalle del grid, hacerlo full-width al final**

Quitar la sección Detalle del grid `md:grid-cols-2` (líneas 54-68). El grid que queda tiene solo el Concepto — convertirlo a full-width:
```diff
-    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
-      <!-- Concepto -->
-      ...
-      <!-- Detalle — eliminar de aquí -->
-    </div>
+    <!-- Concepto — standalone full-width -->
+    <div class="space-y-1">
+      <label class="text-sm font-medium text-foreground">
+        {{ $t('talonarios.concepto') }} <span class="text-destructive">*</span>
+      </label>
+      <Select v-model="concepto">
+        ...
+      </Select>
+      <p v-if="errors.concepto" class="text-xs text-destructive">{{ errors.concepto }}</p>
+    </div>
```

**3. Agregar Detalle después del bloque Total** (antes de las acciones):
```html
<!-- Detalle — al fondo, full-width -->
<div class="space-y-1">
  <label class="text-sm font-medium text-foreground">
    {{ $t('talonarios.detalle') }} <span class="text-destructive">*</span>
  </label>
  <Textarea
    ref="detalleRef"
    v-bind="detalleAttrs"
    v-model="detalle"
    :rows="3"
    :placeholder="$t('talonarios.detalleRequired')"
    class="w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring resize-none"
    :class="errors.detalle ? 'border-destructive' : ''"
  />
  <p v-if="errors.detalle" class="text-xs text-destructive">{{ errors.detalle }}</p>
</div>
```

**4. Script** — quitar la desestructuración de `notes`/`notesAttrs`:
```diff
-const { value: notes, handleChange: notesAttrs } = useField<string>('notes')
```
`notes` sigue en el schema como `optional()` y en `initialValues` como `''` — OK dejar o eliminar.

---

## Item 3 — Form talonario: limitar montos a pendingAmount

### Contexto

`ReceiptCreatePage.vue` ya tiene `admissionContext.value.pendingAmount`. No se pasa a `ReceiptForm`.

### Cambios en `ReceiptCreatePage.vue`

**Agregar `pendingAmount` al `prefill`** (dentro del `onMounted` donde se construye `prefill`):
```diff
     prefill.value = {
       patientName: detail.patientName ?? '',
       patientId: detail.admission.patientId,
       accountStatementId: detail.statement.id,
+      pendingAmount: detail.statement.pendingAmount,
       concepts: [{
         type: 'HOSPITAL',
         description: `Servicios admisión ${detail.admission.admissionNumber}`,
         amount: detail.statement.pendingAmount,
       }],
     }
```

### Cambios en `ReceiptForm.vue`

**1. Añadir `pendingAmount` a la interfaz `Prefill`:**
```diff
 interface Prefill {
   patientName?: string
   patientId?: string
   accountStatementId?: string
+  pendingAmount?: number
   concepts?: Array<{ type: string; description: string; amount: number }>
 }
```

**2. Agregar `maxForConcept` computed** (después de `total`):
```ts
function maxForConcept(idx: number): number | undefined {
  const cap = props.prefill?.pendingAmount
  if (cap == null) return undefined
  const others = fields.value.reduce(
    (s, f, i) => i !== idx ? s + (Number(f.value.amount) || 0) : s,
    0
  )
  return Math.max(0, cap - others)
}
```

**3. En el input de Amount** — agregar `:max` y clamp en input:
```diff
 <Input
   v-model.number="field.value.amount"
   type="number"
   min="0"
   step="0.01"
+  :max="maxForConcept(idx)"
+  @input="(e: Event) => {
+    const max = maxForConcept(idx)
+    const v = Number((e.target as HTMLInputElement).value)
+    if (max != null && v > max) field.value.amount = max
+  }"
   placeholder="0.00"
   class="..."
 />
```

**4. Mostrar advertencia de saldo agotado** (debajo del bloque Total, encima de Detalle):
```html
<p
  v-if="props.prefill?.pendingAmount != null && total > props.prefill.pendingAmount"
  class="text-sm text-destructive"
>
  El total (Q {{ total.toFixed(2) }}) supera el saldo pendiente
  (Q {{ props.prefill.pendingAmount.toFixed(2) }}).
</p>
```

**Edge case**: El cap solo aplica cuando el form viene de una admisión (`prefill.pendingAmount != null`). Si el talonario se crea standalone (sin admisión), no hay límite.

---

## Item 4 — Dialog "Editar cargos" más grande

**`AdmissionDetailPage.vue` línea 521:**
```diff
 <Dialog v-model:open="editPkgOpen">
-  <DialogContent class="max-w-lg">
+  <DialogContent class="max-w-3xl">
```

El `max-w-3xl` (48rem) da suficiente espacio para ver textos de descripción de items y el stepper sin truncamiento.

---

## Item 5 — Estado de Cuenta PDF: columna Subtotal + descripción más ancha

### Situación actual

- `COLS: [usize; 3] = [3, 5, 2]` — compartido entre receipt PDF y EC PDF.
- `ec_section` usa 3 columnas: tipo, descripción, monto.
- El handler en `admission.rs` codifica qty/precio en la descripción:
  ```rust
  format!("{} · {} × Q{:.2}", i.description, q, p)
  ```
- `ConceptPdfData` no tiene campos `quantity`/`unit_price`.

### Cambios en `hospital-belen-api/src/infrastructure/pdf/mod.rs`

**1. Agregar `subtotal_label` a `ConceptPdfData`:**
```diff
 pub struct ConceptPdfData {
     pub concept_type: String,
     pub description: String,
     pub amount: f64,
+    pub subtotal_label: Option<String>,
 }
```

**2. Agregar constante `EC_COLS` para secciones de items (4 columnas):**
```diff
 // ── Column weights ──────────────────────────────────────────────────────────
 const COLS: [usize; 3] = [3, 5, 2];
+// EC item rows: tipo | descripción | subtotal (qty×precio) | monto
+const EC_COLS: [usize; 4] = [2, 7, 3, 2];
```

**3. Cambiar `ec_section` a 4 columnas:**

Firma actual: `fn ec_section(doc, title, rows: &[(String, String, f64)])`

Cambiar a: `fn ec_section(doc, title, rows: &[(String, String, Option<String>, f64)])`

```diff
 fn ec_section(
     doc: &mut Document,
     title: &str,
-    rows: &[(String, String, f64)],
+    rows: &[(String, String, Option<String>, f64)],
 ) -> Result<(), String> {
```

Body actualizado:
```rust
    let mut table = TableLayout::new(EC_COLS.to_vec());
    table.set_cell_decorator(FrameCellDecorator::new(false, true, false));

    let mut subtotal = 0.0;
    for (c1, desc, sub_label, amount) in rows {
        subtotal += *amount;
        let mut row = table.row();
        row.push_element(cp(Paragraph::new(/* c1 */)));
        row.push_element(cp(Paragraph::new(/* desc */)));
        row.push_element(cp(Paragraph::new(genpdf::style::StyledString::new(
            sub_label.as_deref().unwrap_or(""),
            Style::new().with_font_size(8).with_color(MUTED),
        )).aligned(Alignment::Right)));
        row.push_element(cp(Paragraph::new(/* "Q {:.2}" */)));
        row.push().map_err(|e| e.to_string())?;
    }

    // Subtotal row — 4 cols
    {
        let mut row = table.row();
        row.push_element(cp(Paragraph::new(genpdf::style::StyledString::new("", Style::new()))));
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
```

**4. Actualizar tablas de totales EC (grand, pagado, saldo) a 4 columnas**

Cambiar `TableLayout::new(COLS.to_vec())` → `TableLayout::new(EC_COLS.to_vec())` en los tres bloques de totales de `generate_estado_de_cuenta_pdf`. Cada fila ya tiene 3 elementos (vacío, label, monto) — agregar un 4to elemento vacío en col 3:

```diff
-let mut grand = TableLayout::new(COLS.to_vec());
+let mut grand = TableLayout::new(EC_COLS.to_vec());
 {
     let mut row = grand.row();
     row.push_element(/* "" */);
+    row.push_element(/* "" */);
     row.push_element(/* "GRAN TOTAL" */);
     row.push_element(/* "Q x.xx" */);
```

Mismo patrón para `pagado` y `saldo`.

**5. Cambiar el `collect` lambda del handler a incluir subtotal_label:**

En `generate_estado_de_cuenta_pdf` → `let collect = ...` (línea 650):
```diff
-    let collect = |concepts: &[&str]| -> Vec<(String, String, f64)> {
+    let collect = |concepts: &[&str]| -> Vec<(String, String, Option<String>, f64)> {
         data.items
             .iter()
             .filter(|i| concepts.contains(&i.concept_type.as_str()))
             .map(|i| {
                 (
                     concept_label(&i.concept_type).to_string(),
                     i.description.clone(),
+                    i.subtotal_label.clone(),
                     i.amount,
                 )
             })
             .collect()
     };
```

Y el bloque `hosp` de HOSPITALIZACIÓN/EXTRAS:
```diff
-    hosp.push((format!("{} x{}", "Extra", e.quantity), e.description.clone(), e.total));
+    hosp.push((format!("{} x{}", "Extra", e.quantity), e.description.clone(), None, e.total));
```

### Cambios en `hospital-belen-api/src/web/handlers/admission.rs`

**Cambiar el mapeo de items** — dejar la descripción limpia y pasar `subtotal_label`:
```diff
         .map(|i| {
-            let description = match (i.quantity, i.unit_price) {
-                (Some(q), Some(p)) => format!("{} · {} × Q{:.2}", i.description, q, p),
-                _ => i.description.clone(),
-            };
+            let subtotal_label = match (i.quantity, i.unit_price) {
+                (Some(q), Some(p)) => Some(format!("{:.0} × Q{:.2}", q, p)),
+                _ => None,
+            };
             ConceptPdfData {
                 concept_type: i.concept.clone(),
-                description,
+                description: i.description.clone(),
                 amount: i.original_amount,
+                subtotal_label,
             }
         })
```

**Nota**: Los usos de `ConceptPdfData` en el receipt handler (`receipt.rs`) también necesitan el campo `subtotal_label: None` al construirlos.

---

## Usos adicionales de `ConceptPdfData` a actualizar

Buscar todas las instancias donde se construye `ConceptPdfData { ... }` y agregar `subtotal_label: None`:
- `hospital-belen-api/src/web/handlers/receipt.rs` — si construye `ConceptPdfData` directamente.

```bash
grep -rn "ConceptPdfData {" hospital-belen-api/src/
```

---

## Edge cases

- **Item 3**: Si el usuario edita un concepto existente y su valor ya excede el cap (por haber bajado el pendingAmount), el clamp al abrir la pantalla no se aplica automáticamente — solo al interactuar con ese input. Aceptable.
- **Item 5**: Si `quantity`/`unit_price` es `None` para un item (ej. extras de enfermería, conceptos manually added), `subtotal_label = None` → col3 vacía. Correcto.
- **Item 5**: Extras (`ExtraPdfLine`) en la sección HOSPITALIZACIÓN/EXTRAS no tienen `subtotal_label` — se pasa `None`. OK, su `quantity × unit_price` ya no se muestra en descripción.
