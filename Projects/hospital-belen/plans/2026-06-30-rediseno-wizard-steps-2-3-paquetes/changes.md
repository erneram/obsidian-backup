# Changes — estado-cuenta wizard: UX rediseño steps 2 y 3

**Implementado:** 2026-06-30  
**Modo:** stage-2

---

## Archivos modificados

### `types/index.ts`
- `PendingPackageItem` — añade `excluded?: boolean`

### `StepItems.vue`
- `toggleExcludeItem(pkgIdx, itemIdx)` — toggle `item.excluded`
- Fila de ítem en paquete pendiente: `opacity-50`/`line-through` si excluido; `[- N +]` deshabilitados si excluido; total muestra `Q0.00` si excluido; Trash2 togglea exclusión (rojo si activo, muted si excluido)

### `StepReview.vue`
- Removidos botones "Exportar Excel" y "Generar PDF" (e imports `FileDown`, `FileSpreadsheet`, `ref`)
- Card "Paquete por aplicar": informacional, sin botón de acción; ítems excluidos con `opacity-50` + `line-through` + "—" en columna de qty
- `pendingPackageTotal` computed: suma `qty × unitPrice` de ítems no excluidos en `pendingPackages`
- Total paquete muestra `totalPackage + pendingPackageTotal`
- `applying` ref y `applyPending()` eliminados (ya no aplica en step 3)

### `EstadoCuentaWizardPage.vue`
- `onFinish` → async; itera `pendingPackages`, filtra ítems excluidos, skipea si `items.length === 0`, llama `ec.applyPackage` por cada paquete con ítems
- Botón "VER" deshabilitado con `:disabled="ec.loading.value"` mientras aplica

---

## Qué revisar (Tester)

- Step 2: Trash2 por ítem tacha el ítem (toggle), `[- N +]` se deshabilitan, total = Q0.00
- Step 2: Trash2 en header sigue eliminando el paquete completo
- Step 3: ítems excluidos visibles con tachado y "—" en qty; sin botón "Aplicar paquete"
- Step 3: totalPaquete refleja pendiente (no-excluidos) + ya aplicados en server
- "VER": aplica solo ítems no excluidos; si todos excluidos, `applyPackage` no se llama
- Navegar step 3 → back → step 3: exclusiones persisten
