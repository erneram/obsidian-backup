# Test Results — AdmissionDetailPage: Rediseño secciones CARGOS y EXTRAS

**Ejecutado:** 2026-07-01  
**Modo:** stage-3 / stage-2  
**Resultado:** ✅ PASS

---

## Verificación

### Sección CARGOS → "Cargos de Inventario"
- Card título: `"Cargos de Inventario"` ✅ (línea 130)
- Tabla itera `extras` ✅ (línea 151: `v-for="extra in extras"`)
- Trash llama `confirmRemoveExtra(extra.id)` ✅ (línea 162)

### Dialog "Agregar Cargo de Inventario"
- `addInventoryChargeOpen` ref ✅ (línea 859)
- Combobox Bodega: `Popover` + `CommandInput` + carga única (`invStoragesLoaded`) ✅ (líneas 412, 861, 883–886)
- Combobox Ítem: solo visible tras seleccionar bodega (`v-if="invSelectedStorage"`) ✅ (línea 436)
- Fila con nombre/precio editable/`[- qty +]`/total ✅ (línea 473)
- Submit `submitInventoryCharge` ✅ (línea 496)

### Sección EXTRAS DE ENFERMERÍA
- `conceptualItems` computed filtra `concept !== 'PACKAGE' && concept !== 'OTROS'` ✅ (líneas 854–856)
- Tabla itera `conceptualItems` ✅ (línea 285)
- Trash llama `confirmRemoveItem(item.id)` ✅ (línea 285)

### Eliminado
- `addItemOpen`, `itemForm`, `openAddItem`, `submitAddItem` — 0 hits en archivo ✅

### Imports
- `Popover*`, `Command*`, `ChevronsUpDown`, `inventoryService`, `InventoryItem` presentes ✅

## Estado del sistema
HMR activo en `AdmissionDetailPage.vue`, sin errores de compilación.
