VERDICT: SHIP

## Razonamiento

5 items implementados conforme al spec. `EC_COLS` 4-columnas correcto en todas las rutas, `maxForConcept` lógica sana, `receipt.rs` no requería cambio (usa `From<&Receipt>` impl que ya tiene `subtotal_label: None`). Dos observaciones no bloqueantes.

## Hallazgos

🟢 BAJO: `@page { margin: 0 }` + `padding: 10mm; box-sizing: border-box` en ReceiptPreview.vue ✅
🟢 BAJO: `notesAttrs` eliminado. `notes` permanece en `initialValues`/`values.notes` (campo hidden, se envía como `''` a la API) — comportamiento correcto: no rompió el submit ✅
🟢 BAJO: `maxForConcept` — `Math.max(0, cap - others)` correcto. `:max` nativo + clamp reactivo en `@input` doble-guarda. vee-validate re-renderiza tras `field.value.amount = max` ✅
🟢 BAJO: `receipt.rs` — 0 instancias de `ConceptPdfData { }` directas; construye mediante `From<&Receipt>` en pdf/mod.rs (l.84: `subtotal_label: None`). Spec tenía la cobertura en el lugar correcto ✅
🟢 BAJO: `grand`/`pagado`/`saldo` tienen 4 `push_element` por fila (2 vacíos + label + amount), alineado con `EC_COLS: [2, 7, 3, 2]` ✅
🟡 BAJO: `ec_section` agrega fila "Subtotal" al fondo de CADA sección. Para secciones de 1 item el subtotal == el monto del item (redundante visualmente). No es un bug — spec lo define explícitamente.
🟡 BAJO: `notes` enviado como `''` al crear un talonario (campo no visible en form). Si la columna `receipts.notes` tiene constraint NOT NULL + default `''`, es seguro; si es nullable preferible enviar `null`. No bloquea.
