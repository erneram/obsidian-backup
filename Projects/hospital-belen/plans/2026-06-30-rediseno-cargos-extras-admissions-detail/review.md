VERDICT: SHIP

## Razonamiento

Swap CARGOS↔EXTRAS implementado correctamente en único archivo. Nuevo flujo de inventario wired al endpoint correcto (`createExtra` con `inventoryItemId`). Eliminaciones confirmadas. Sin scope leakage.

## Hallazgos

### Sección CARGOS → "Cargos de Inventario"
🟢 BAJO: Tabla itera `extras` (línea 151); Trash llama `confirmRemoveExtra` → `admissionService.deleteExtra` (restaura stock) ✅

🟢 BAJO: Dialog inventario — `invStoragesLoaded` lazy-load correcto; debounce 300ms en `invSearchTimer` (clearTimeout + setTimeout); `listAllStorages()` y `listInventory()` existen en `inventoryService` ✅

🟢 BAJO: `submitInventoryCharge` envía `inventoryItemId` + `storageId` + `description` + `quantity` + `unitPrice` → `createExtra`; refresca con `Promise.all([fetchAdmission, loadExtras])` ✅

### Sección EXTRAS DE ENFERMERÍA
🟢 BAJO: `conceptualItems` filtra `concept !== 'PACKAGE' && concept !== 'OTROS'` (líneas 854-856) ✅

🟢 BAJO: `submitAddExtra` usa `adm.addItem` con `concept`/`description`/`amount`; validaciones previas correctas; `confirmRemoveItem` llama `adm.removeItem` ✅

### Eliminaciones
🟢 BAJO: `addItemOpen`, `itemForm`, `openAddItem`, `submitAddItem`, `itemError` — 0 referencias en archivo ✅

## Sin hallazgos de seguridad, commits no pedidos, ni cambios fuera de scope.
