# Changes — AdmissionDetailPage: Rediseño secciones CARGOS y EXTRAS

**Implementado:** 2026-06-30  
**Modo:** stage-2

---

## Archivos modificados

### `AdmissionDetailPage.vue`

**Sección CARGOS → "Cargos de Inventario"**
- Tabla ahora itera `extras` (ítems de bodega): columnas Descripción | Cant. | Precio Unit. | Total
- Trash llama `confirmRemoveExtra(extra.id)` (restaura inventario)
- Botón "Agregar cargo" → abre `addInventoryChargeOpen` dialog

**Nuevo Dialog "Agregar Cargo de Inventario"**
- Combobox Bodega (carga única `invStoragesLoaded`, filtro por nombre)
- Combobox Ítem con búsqueda debounced 300ms (visible solo tras seleccionar bodega)
- Fila seleccionada: nombre | precio editable | `[- qty +]` | total
- Submit llama `admissionService.createExtra` con `inventoryItemId` + `storageId` → descuenta inventario
- Refresca `adm.fetchAdmission` + `loadExtras` en éxito

**Sección EXTRAS DE ENFERMERÍA**
- Tabla ahora itera `conceptualItems` computed (`detail.items` filtrado: no PACKAGE, no OTROS)
- Columnas: Concepto | Descripción | Monto
- Trash llama `confirmRemoveItem(item.id)`
- Removed `extrasLoading` spinner (data viene de `detail`)

**Dialog "Agregar Extra de Enfermería" — reemplazado**
- Ahora: Select Concepto + Input Descripción + Input Monto
- Submit llama `adm.addItem` (crea `account_statement_item`) + refresca `adm.fetchAdmission`

**Eliminado**
- `addItemOpen`, `itemError`, `itemForm`, `openAddItem`, `submitAddItem` (antigua lógica de "Agregar Cargo")

**Imports agregados**
- `Popover`, `PopoverTrigger`, `PopoverContent`, `Command`, `CommandInput`, `CommandEmpty`, `CommandGroup`, `CommandItem` de `@inksightdev/ui`
- `Minus`, `ChevronsUpDown` de `lucide-vue-next`
- `inventoryService`, `Storage`, `InventoryItem` de `@/modules/inventory`

---

## Qué revisar (Tester)

- "Agregar cargo" → dialog bodega/ítem; buscar ítem; seleccionar; ajustar qty/precio; Agregar → aparece en tabla Cargos de Inventario con descuento de stock
- Trash en fila de cargo → confirm → elimina y restaura stock en bodega
- "Agregar Extra" → dialog con Concepto/Descripción/Monto → aparece en tabla Extras de Enfermería
- Trash en Extra → confirm → elimina `account_statement_item`
- `conceptualItems` no muestra ítems PACKAGE ni OTROS
- Saldo del header se actualiza tras cada operación
