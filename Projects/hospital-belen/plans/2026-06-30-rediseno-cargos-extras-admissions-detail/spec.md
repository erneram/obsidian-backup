# Spec — AdmissionDetailPage: Rediseño secciones CARGOS y EXTRAS
**project:** hospital-belen  
**dispatch:** stage-1 (2026-07-01)

---

## Contexto técnico

### Endpoints relevantes

| Endpoint | Qué hace |
|----------|----------|
| `POST /admissions/{id}/items` | Crea un `account_statement_item` por concepto/descripción/monto. Sin inventario. |
| `POST /admissions/{id}/extras` | Crea un `admission_extra` **y** un `account_statement_item` ligado (concept=OTROS). Si `inventoryItemId` está presente, **descuenta el inventario** en la misma transacción. |
| `DELETE /admissions/{id}/extras/{eid}` | Elimina el extra **y** restaura la cantidad de inventario si tenía `inventoryItemId`. |

### Estado actual de las secciones

- **CARGOS** (`detail.items`): todos los `account_statement_items`; el diálogo "Agregar cargo" usa `addItem` (concepto + descripción + monto).
- **EXTRAS DE ENFERMERÍA** (`extras`): todos los `AdmissionExtra`; el diálogo usa `createExtra` (descripción + qty + precio).

---

## OPEN QUESTION — RESUELTA (2026-07-01)

**Decisión del humano**: mantener los paquetes aplicados visibles exactamente como están ahora. Además, **hacer el swap** entre las dos secciones:
- Lo que antes era "CARGOS" (conceptos: Hospitalización, Laboratorios, etc.) → pasa a ser la sección de "EXTRAS DE ENFERMERÍA".
- Lo que antes era "EXTRAS DE ENFERMERÍA" (ítems de bodega) → pasa a ser la sección de "CARGOS".
- Los paquetes aplicados (`concept = 'PACKAGE'`) se mantienen visibles en su card actual sin cambios.

---

## Cambios

### Único archivo: `AdmissionDetailPage.vue`

---

### 1. Sección CARGOS — tabla + diálogo

#### 1a. Tabla: cambiar fuente de `detail.items` a `extras`

Reemplazar las columnas de la tabla de Cargos (actualmente: Concepto | Descripción | Monto) por las columnas de `AdmissionExtra`:

```
Descripción | Cant. | Precio Unit. | Total | [🗑️]
```

Iterar sobre `extras` (ya cargado por `loadExtras()`).  
El Trash llama a `confirmRemoveExtra(extra.id)` (ya existe; restaura inventario).

Header del card: renombrar título a `"Cargos de Inventario"` o mantener `"Cargos"` — a criterio del coder.

#### 1b. Diálogo "Agregar cargo" — REEMPLAZAR por selector de inventario

Eliminar el Dialog actual de "Agregar Cargo" (concepto/descripción/monto).  
Reemplazar por un Dialog con el siguiente flujo:

```
1. Combobox — Seleccionar bodega
2. Combobox — Buscar ítem (sólo visible tras seleccionar bodega, búsqueda debounced 300ms)
3. Tras seleccionar ítem:
   - Fila: [Nombre del producto] | Q[unit price] | [- qty +] | Q[total]
   - Unit price editable (Input type=number)
4. Botón "Agregar"
```

**Estado local del diálogo** (todos `ref` / `reactive` dentro del script, no en composable):

```ts
const addInventoryChargeOpen = ref(false)
const invStorages = ref<Storage[]>([])
const invStoragesLoaded = ref(false)           // ponytail: cargar una vez
const invSelectedStorage = ref<Storage | null>(null)
const invStorageQuery = ref('')
const invStoragePopoverOpen = ref(false)
const invItemQuery = ref('')
const invItemPopoverOpen = ref(false)
const invItemResults = ref<InventoryItem[]>([])
const invSelectedItem = ref<InventoryItem | null>(null)
const invQty = ref(1)
const invUnitPrice = ref(0)
const invError = ref<string | null>(null)
const invLoading = ref(false)
let invSearchTimer: ReturnType<typeof setTimeout> | null = null
```

**Funciones**:

```ts
async function openAddInventoryCharge() {
  // cargar bodegas una vez
  if (!invStoragesLoaded.value) {
    const res = await inventoryService.listAllStorages()
    if (res.success && res.data?.data) invStorages.value = res.data.data
    invStoragesLoaded.value = true
  }
  invSelectedStorage.value = null
  invSelectedItem.value = null
  invItemQuery.value = ''
  invItemResults.value = []
  invQty.value = 1
  invUnitPrice.value = 0
  invError.value = null
  addInventoryChargeOpen.value = true
}

function onInvItemSearchInput() {
  if (invSearchTimer) clearTimeout(invSearchTimer)
  invSearchTimer = setTimeout(async () => {
    if (!invSelectedStorage.value || !invItemQuery.value.trim()) {
      invItemResults.value = []
      return
    }
    const res = await inventoryService.listInventory({
      storageId: invSelectedStorage.value.id,
      productName: invItemQuery.value,
    })
    invItemResults.value = res.success && res.data?.data ? res.data.data : []
  }, 300)
}

function onInvItemSelect(item: InventoryItem) {
  invSelectedItem.value = item
  invUnitPrice.value = item.defaultPrice
  invQty.value = 1
  invItemPopoverOpen.value = false
}

async function submitInventoryCharge() {
  if (!invSelectedItem.value || !invSelectedStorage.value) {
    invError.value = 'Selecciona una bodega e ítem'
    return
  }
  invLoading.value = true
  invError.value = null
  try {
    const res = await admissionService.createExtra(route.params.id as string, {
      inventoryItemId: invSelectedItem.value.id,
      storageId: invSelectedStorage.value.id,
      description: invSelectedItem.value.productName,
      quantity: invQty.value,
      unitPrice: invUnitPrice.value,
    })
    if (res.success) {
      addInventoryChargeOpen.value = false
      await Promise.all([
        adm.fetchAdmission(route.params.id as string),
        loadExtras(),
      ])
    } else {
      invError.value = 'Error al agregar cargo'
    }
  } finally {
    invLoading.value = false
  }
}
```

**Imports a agregar**:
```ts
import { inventoryService } from '@/modules/inventory/services/inventoryService'
import type { Storage, InventoryItem } from '@/modules/inventory/types'
```

---

### 2. Sección EXTRAS DE ENFERMERÍA — tabla + diálogo

#### 2a. Tabla: cambiar fuente de `extras` a `detail.items` filtrado

Agregar computed:
```ts
const conceptualItems = computed(
  () => detail.value?.items.filter(i => i.concept !== 'PACKAGE' && i.concept !== 'OTROS') ?? []
)
```

Cambiar la tabla de EXTRAS DE ENFERMERÍA para iterar sobre `conceptualItems` en vez de `extras`.  
Columnas: Concepto | Descripción | Monto | [🗑️]  
Trash llama a `confirmRemoveItem(item.id)` (ya existe).

#### 2b. Diálogo "Agregar Extra" — REEMPLAZAR con el form conceptual

Eliminar el Dialog actual de "Agregar Extra de Enfermería" (descripción + qty + precio).  
Reemplazar por la estructura del Dialog actual de "Agregar Cargo" (concepto + descripción + monto):

```html
<Dialog v-model:open="addExtraOpen">
  <DialogContent class="max-w-md">
    <DialogHeader>
      <DialogTitle>Agregar Extra de Enfermería</DialogTitle>
    </DialogHeader>
    <form class="space-y-4" @submit.prevent="submitAddExtra">
      <!-- Concepto select (igual al actual addItem dialog) -->
      <div class="space-y-1">
        <Label>Concepto *</Label>
        <Select v-model="extraForm.concept" required>
          <SelectTrigger class="w-full">
            <SelectValue placeholder="Seleccionar concepto..." />
          </SelectTrigger>
          <SelectContent>
            <SelectItem v-for="c in CONCEPT_OPTIONS" :key="c" :value="c">
              {{ conceptLabel(c) }}
            </SelectItem>
          </SelectContent>
        </Select>
      </div>
      <div class="space-y-1">
        <Label>Descripción *</Label>
        <Input v-model="extraForm.description" placeholder="Descripción del cargo" required />
      </div>
      <div class="space-y-1">
        <Label>Monto (Q) *</Label>
        <Input v-model="extraForm.amountStr" type="number" step="0.01" min="0.01" required />
      </div>
      <div v-if="extraError" class="text-destructive text-sm">{{ extraError }}</div>
      <DialogFooter>
        <Button variant="outline" type="button" @click="addExtraOpen = false">Cancelar</Button>
        <Button type="submit" :disabled="extrasLoading">
          {{ extrasLoading ? 'Guardando...' : 'Agregar' }}
        </Button>
      </DialogFooter>
    </form>
  </DialogContent>
</Dialog>
```

Actualizar `extraForm` reactive y `submitAddExtra`:

```ts
// Reemplazar extraForm (actualmente: description, quantity, unitPrice)
const extraForm = reactive({ concept: '', description: '', amountStr: '' })

async function submitAddExtra() {
  const amount = parseFloat(extraForm.amountStr)
  if (!extraForm.concept) { extraError.value = 'Concepto requerido'; return }
  if (!extraForm.description.trim()) { extraError.value = 'Descripción requerida'; return }
  if (!amount || amount <= 0) { extraError.value = 'Monto inválido'; return }
  extrasLoading.value = true
  extraError.value = null
  try {
    const ok = await adm.addItem(route.params.id as string, {
      concept: extraForm.concept,
      description: extraForm.description.trim(),
      amount,
    })
    if (ok) {
      addExtraOpen.value = false
      await adm.fetchAdmission(route.params.id as string)
    } else {
      extraError.value = adm.error.value ?? 'Error al agregar'
    }
  } finally {
    extrasLoading.value = false
  }
}
```

Y `openAddExtra`:
```ts
function openAddExtra() {
  extraForm.concept = ''
  extraForm.description = ''
  extraForm.amountStr = ''
  extraError.value = null
  addExtraOpen.value = true
}
```

---

### 3. Eliminar el Dialog antiguo "Agregar Cargo" (concepto/descripción/monto)

El Dialog de `addItemOpen` (lines ~444-488) ya no es necesario — su funcionalidad se movió a EXTRAS.  
Eliminar también: `addItemOpen`, `itemError`, `itemForm`, `openAddItem`, `submitAddItem`, `confirmRemoveItem` migra a llamar directo desde el nuevo botón de EXTRAS.

Verificar que `confirmRemoveItem` siga usándose para el trash en `conceptualItems` — si es así, mantenerlo.

---

## Resumen de flujo resultante

| Sección | Tabla fuente | Agregar via | Delete via |
|---------|-------------|-------------|-----------|
| **Cargos** (arriba) | `extras` | `createExtra` + inventoryItemId (descuenta stock) | `deleteExtra` (restaura stock) |
| **Extras de Enfermería** (abajo) | `detail.items.filter(non-PACKAGE, non-OTROS)` | `addItem` con concepto | `removeItem` |
| *(Paquetes aplicados — OPEN QUESTION)* | `detail.items.filter(PACKAGE)` | via "Aplicar Paquete" existente | `removeItem` |

---

## Archivos a modificar

- `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue` — único archivo
