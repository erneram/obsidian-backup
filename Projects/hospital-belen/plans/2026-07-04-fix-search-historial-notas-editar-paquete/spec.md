# Spec — 3 features: search fix real, historial pagos, editar paquete
**project:** hospital-belen  
**dispatch:** stage-1 (2026-07-05, effort=high)

---

## Archivos a modificar

| Archivo | Feature |
|---------|---------|
| `inksight-web-components/packages/ui-brand/src/components/CommandItem.vue` | 1 |
| `hospital-belen-web/kairosaid/src/modules/estado-cuenta/components/StepItems.vue` | 1 |
| `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue` | 1, 3 |
| `hospital-belen-api/src/domain/receipt/mod.rs` | 2 |
| `hospital-belen-api/src/infrastructure/database/repositories/receipt_repository.rs` | 2 |
| `hospital-belen-api/src/infrastructure/pdf/mod.rs` | 2 |
| `hospital-belen-web/kairosaid/src/modules/receipts/types/index.ts` | 2 |

---

## Feature 1 — Search: root cause real

### Traza del flujo completo

**StepItems.vue — item combobox:**
1. Usuario abre el Popover, escribe "Asp" en `CommandInput`
2. `CommandInput.onInput` fires:
   - `emit('update:modelValue', 'Asp')` → `itemQuery.value = 'Asp'` (v-model) + `onItemSearchInput()` (@update:model-value)
   - `search.value = 'Asp'` ← **AQUÍ está el problema**
3. `onItemSearchInput()` setea debounce 300ms → API call → `inventoryResults.value = [items...]`
4. `filteredInventoryResults` computed → items filtered ✓
5. Vue re-renderiza `<CommandGroup>`:
   ```html
   <CommandItem v-for="item in filteredInventoryResults" :value="item.id" ...>
   ```
6. **`CommandItem.visible`**: `item.id.toLowerCase().includes('asp')` → `false` para todos los UUIDs
7. Todos los `<CommandItem>` se ocultan con `v-if="visible"` — el usuario no ve nada

**La API funciona correctamente.** Los resultados llegan a `inventoryResults`. El problema está en el filtro interno de `CommandItem`.

### Root cause confirmado

`CommandItem.vue` (en `inksight-web-components`):
```ts
const search = inject<Ref<string>>('command-search')
const visible = computed(() => {
  if (!search?.value || !props.value) return true
  return props.value.toLowerCase().includes(search.value.toLowerCase())
})
```

Filtra SIEMPRE por `props.value` (el UUID). `Command.vue` NO tiene prop `shouldFilter` — el
`:shouldFilter="false"` que pusimos antes es silenciosamente ignorado.

Cuando user escribe "Asp":
- `Command`'s `search.value = 'Asp'` (vía `CommandInput.onInput`)
- `CommandItem.visible = 'abc-uuid-123'.includes('asp')` = `false`
- Todos los items se ocultan

### Fix

**`CommandItem.vue` línea 34-37** — agregar guard para search vacío OR usar el valor para
filtrar, pero como el filtrado ya lo hacen los computeds externos, la solución más limpia
es cambiar `:value` en los `<CommandItem>` de UUID a texto visible:

**Opción A (recomendada) — cambiar `:value` en los comboboxes:**

`CommandItem` filtra `props.value.includes(search)`. Si `:value` es el nombre del item,
el filtro coincide con lo que escribe el usuario. Los `@select` handlers usan closures y
NO dependen del valor emitido.

**`StepItems.vue`:**
```diff
// línea 27 — package items
-:value="p.id"
+:value="p.name"

// línea 152 — storages
-:value="s.id"
+:value="s.name"

// línea 185 — inventory items
-:value="item.id"
+:value="item.productName"
```

**`AdmissionDetailPage.vue` — modal "Agregar Cargo de Inventario":**
```diff
// storage CommandItem
-:value="s.id"
+:value="s.name"

// inventory item CommandItem
-:value="item.id"
+:value="item.productName"
```

**Opción B — modificar `CommandItem.vue` directamente:**

Agregar prop `shouldFilter?: boolean` (default true) y saltar el filtro si `false`:
```diff
+const props = defineProps<{ value?: string; disabled?: boolean; class?: string; shouldFilter?: boolean }>()
 const visible = computed(() => {
+  if (props.shouldFilter === false) return true
   if (!search?.value || !props.value) return true
   return props.value.toLowerCase().includes(search.value.toLowerCase())
 })
```

Luego pasar `shouldFilter=false` en todos los `<CommandItem>` de comboboxes con búsqueda server-side.

**Diferencia entre A y B:**
- A: 5 líneas en 2 archivos frontend. Sin cambios a la librería.
- B: 1 línea en la librería + 5 `:shouldFilter="false"` props. Más correcto arquitecturalmente.

Si la librería ya tiene versión bump pendiente → usar B (agrupa con el cambio de Sonner). Si no → usar A.

**Para el `filteredPackages` / `filteredStorages` / `filteredInventoryResults` computeds:**
Con la Opción A, la condición del Command filter coincide con la del computed (ambos filtran
por nombre). El resultado es el mismo — no hace falta quitar los computeds. Con Opción B,
el Command no filtra y el computed lo hace todo. Ambos funcionan.

---

## Feature 2 — Historial de Pagos: Notas + Por + PDF talonario

### Diagnóstico actual

**Notas column:** `historyRows` mapea receipts con `notes: r.concepto ?? null`. El campo
`concepto` almacena el tipo de concepto (ej. "HOSPITALIZACION"), no el texto libre del
usuario. El texto libre está en `r.detalle` (columna `receipts.detalle` en DB).

**Por column:** `by: null` hardcoded. El `Receipt` struct tiene `created_by: Uuid` pero
no `created_by_name`. Necesita JOIN con `users` (que tiene `first_name`, `last_name`).

### Cambios backend

#### `hospital-belen-api/src/domain/receipt/mod.rs`

**1. `ReceiptRow`** — agregar campo (línea después de `created_by`):
```diff
 pub created_by: Uuid,
+pub created_by_name: Option<String>,
 pub created_at: String,
```

**2. `Receipt`** — agregar campo:
```diff
 #[serde(rename = "createdBy")]
 pub created_by: Uuid,
+#[serde(rename = "createdByName")]
+pub created_by_name: Option<String>,
 #[serde(rename = "createdAt")]
```

#### `hospital-belen-api/src/infrastructure/database/repositories/receipt_repository.rs`

**3. List query** (línea ~83) — agregar JOIN y campo:
```diff
-r#"SELECT r.id, r.tenant_id, r.receipt_number, r.patient_id, r.patient_name,
-          r.total_amount::float8, r.status::text AS status,
-          r.void_reason, r.concepto::text AS concepto, r.detalle,
-          r.created_by, r.created_at::text AS created_at,
-          s.admission_id
-   FROM receipts r
-   LEFT JOIN account_statements s ON s.id = r.account_statement_id
+r#"SELECT r.id, r.tenant_id, r.receipt_number, r.patient_id, r.patient_name,
+          r.total_amount::float8, r.status::text AS status,
+          r.void_reason, r.concepto::text AS concepto, r.detalle,
+          r.created_by, r.created_at::text AS created_at,
+          s.admission_id,
+          (u.first_name || ' ' || u.last_name) AS created_by_name
+   FROM receipts r
+   LEFT JOIN account_statements s ON s.id = r.account_statement_id
+   LEFT JOIN users u ON u.id = r.created_by
```

**4. Get query** (línea ~163) — mismo JOIN y campo:
```diff
-r#"SELECT r.id, r.tenant_id, r.receipt_number, r.patient_id, r.patient_name,
-          r.total_amount::float8, r.status::text AS status, r.void_reason,
-          r.concepto::text AS concepto, r.detalle,
-          r.created_by, r.created_at::text AS created_at,
-          s.admission_id
-   FROM receipts r
-   LEFT JOIN account_statements s ON s.id = r.account_statement_id
-   WHERE r.id = $1 AND r.tenant_id = $2
+r#"SELECT r.id, r.tenant_id, r.receipt_number, r.patient_id, r.patient_name,
+          r.total_amount::float8, r.status::text AS status, r.void_reason,
+          r.concepto::text AS concepto, r.detalle,
+          r.created_by, r.created_at::text AS created_at,
+          s.admission_id,
+          (u.first_name || ' ' || u.last_name) AS created_by_name
+   FROM receipts r
+   LEFT JOIN account_statements s ON s.id = r.account_statement_id
+   LEFT JOIN users u ON u.id = r.created_by
+   WHERE r.id = $1 AND r.tenant_id = $2
```

**5. `row_to_receipt()` / `build_receipt()`** — mapear el campo en `Receipt { ... }`:
```diff
 created_by: row.created_by,
+created_by_name: row.created_by_name,
```

#### `hospital-belen-api/src/infrastructure/pdf/mod.rs`

**6. `From<&Receipt>` impl** (línea 84) — usar `detalle` en lugar de `None`:
```diff
-notes: None,
+notes: r.detalle.clone(),
```

**7. `generate_receipt_pdf`** — mover el bloque "NOTES" de después del TOTAL a después
del nombre del paciente (líneas ~248-251). Cambiar "Nota: " por "Detalle: ":

Mover este bloque (actualmente en líneas 331-341) para que quede ANTES del `hrule` que
separa patient de concepts (después de línea 248 `doc.push(Break::new(0.6))`):

```rust
// Insertar después del patient block (línea 249), antes de hrule:
if let Some(notes) = &data.notes
    && !notes.is_empty()
{
    doc.push(Paragraph::new(genpdf::style::StyledString::new(
        format!("Detalle: {}", notes),
        Style::new().with_font_size(9).with_color(MUTED),
    )));
    doc.push(Break::new(0.3));
}
```

Y eliminar el bloque "NOTES" original de después del TOTAL.

### Cambios frontend

**`receipts/types/index.ts`** — agregar `createdByName` a `Receipt`:
```diff
 concepto?: string | null
 detalle?: string | null
+createdByName?: string | null
 admissionId?: string | null
```

**`AdmissionDetailPage.vue`** — `historyRows` computed, sección de receipts:
```diff
 notes: r.concepto ?? null,
+// notas => texto libre del talonario, truncado a 60 chars
 by: null,
```
Cambiar a:
```ts
notes: r.detalle
  ? r.detalle.slice(0, 60) + (r.detalle.length > 60 ? '...' : '')
  : (r.concepto ?? null),
by: r.createdByName ?? null,
```

---

## Feature 3 — Editar paquete + quitar "Aplicar Paquete"

### Quitar botón "Aplicar Paquete"

**`AdmissionDetailPage.vue` header (líneas ~44-47):**
```diff
-<Button
-  v-if="detail.statement.status !== 'CLOSED'"
-  variant="outline"
-  size="sm"
-  @click="openApplyPackage"
->
-  Aplicar Paquete
-</Button>
```

(El modal `applyPkgOpen` y sus variables pueden quedar pero el botón de entrada se elimina.
Si el coder quiere limpiar las variables del apply package modal también, puede hacerlo.)

### Agregar botón "Editar" en card "Paquetes Aplicados"

En el `CardHeader` de "Paquetes Aplicados", agregar junto al "Agregar cargo":
```html
<Button
  v-if="detail.statement.status !== 'CLOSED' && (packageItems.length || inventoryChargeItems.length)"
  size="sm"
  variant="outline"
  @click="openEditPackage"
>
  Editar
</Button>
```

### Nuevo estado para el dialog "Editar cargos"

Agregar en `<script setup>`:

```ts
interface EditableItem {
  id: string
  concept: 'PACKAGE' | 'OTHER'
  description: string
  quantity: number
  unitPrice: number | null
  deleted: boolean
}
const editPkgOpen = ref(false)
const editableItems = ref<EditableItem[]>([])
const editPkgLoading = ref(false)
const editPkgError = ref<string | null>(null)

function openEditPackage() {
  editableItems.value = [
    ...packageItems.value.map(i => ({
      id: i.id,
      concept: 'PACKAGE' as const,
      description: i.description,
      quantity: i.quantity ?? 1,
      unitPrice: i.unitPrice ?? null,
      deleted: false,
    })),
    ...inventoryChargeItems.value.map(i => ({
      id: i.id,
      concept: 'OTHER' as const,
      description: i.description,
      quantity: i.quantity ?? 1,
      unitPrice: i.unitPrice ?? null,
      deleted: false,
    })),
  ]
  editPkgError.value = null
  editPkgOpen.value = true
}

async function submitEditPackage() {
  editPkgLoading.value = true
  editPkgError.value = null
  const admId = route.params.id as string
  try {
    // 1. Delete all marked items
    for (const item of editableItems.value.filter(i => i.deleted)) {
      await admissionService.removeItem(admId, item.id)
    }

    // 2. Inventory items (OTHER) not deleted: bulk replace with new quantities
    const liveInventory = editableItems.value.filter(i => !i.deleted && i.concept === 'OTHER')
    await admissionService.replaceInventoryItems(admId, liveInventory.map(i => ({
      concept: i.concept,
      description: i.description,
      amount: i.quantity * (i.unitPrice ?? 0),
      quantity: i.quantity,
      unitPrice: i.unitPrice ?? undefined,
    })))

    // 3. Package items (PACKAGE) with changed qty: delete original + re-add
    // ponytail: only if unitPrice is known; otherwise qty change is silently skipped
    const pkgOriginalQty = new Map(packageItems.value.map(i => [i.id, i.quantity ?? 1]))
    for (const item of editableItems.value.filter(i => !i.deleted && i.concept === 'PACKAGE')) {
      const origQty = pkgOriginalQty.get(item.id) ?? 1
      if (item.quantity !== origQty && item.unitPrice != null) {
        await admissionService.removeItem(admId, item.id)
        await admissionService.addItem(admId, {
          concept: 'PACKAGE',
          description: item.description,
          amount: item.quantity * item.unitPrice,
        })
      }
    }

    await adm.fetchAdmission(admId)
    await loadExtras()
    editPkgOpen.value = false
  } catch {
    editPkgError.value = 'Error al guardar cambios'
  } finally {
    editPkgLoading.value = false
  }
}
```

### Template del dialog "Editar cargos"

```html
<Dialog v-model:open="editPkgOpen">
  <DialogContent class="max-w-lg">
    <DialogHeader>
      <DialogTitle>Editar cargos</DialogTitle>
    </DialogHeader>
    <div class="space-y-2 py-2 max-h-96 overflow-y-auto">
      <div
        v-for="(item, idx) in editableItems"
        :key="item.id"
        class="flex items-center gap-2 px-3 py-2 rounded-md border text-sm"
        :class="item.deleted ? 'opacity-40' : ''"
      >
        <span class="flex-1 truncate" :class="item.deleted ? 'line-through' : ''">
          {{ item.description }}
          <span class="text-xs text-muted-foreground ml-1">
            {{ item.concept === 'PACKAGE' ? '(paquete)' : '(bodega)' }}
          </span>
        </span>
        <!-- Stepper — desactivado si está borrado -->
        <div class="flex items-center gap-1" v-if="!item.deleted">
          <Button type="button" variant="ghost" size="icon" class="h-7 w-7"
            :disabled="item.quantity <= 1"
            @click="item.quantity = Math.max(1, item.quantity - 1)"
          ><Minus class="h-3 w-3" /></Button>
          <span class="w-8 text-center tabular-nums">{{ item.quantity }}</span>
          <Button type="button" variant="ghost" size="icon" class="h-7 w-7"
            @click="item.quantity++"
          ><Plus class="h-3 w-3" /></Button>
        </div>
        <span class="font-mono w-20 text-right text-xs" v-if="!item.deleted && item.unitPrice != null">
          Q {{ (item.quantity * item.unitPrice).toFixed(2) }}
        </span>
        <!-- Trash: toggle deleted -->
        <Button
          type="button" variant="ghost" size="icon" class="h-7 w-7"
          :class="item.deleted ? 'text-muted-foreground' : 'text-destructive hover:text-destructive'"
          @click="item.deleted = !item.deleted"
        ><Trash2 class="h-3.5 w-3.5" /></Button>
      </div>
      <p v-if="editableItems.length === 0" class="py-4 text-center text-muted-foreground text-sm">
        Sin items editables.
      </p>
    </div>
    <div v-if="editPkgError" class="text-destructive text-sm">{{ editPkgError }}</div>
    <DialogFooter>
      <Button variant="outline" @click="editPkgOpen = false">Cancelar</Button>
      <Button :disabled="editPkgLoading" @click="submitEditPackage">
        {{ editPkgLoading ? 'Guardando...' : 'Guardar' }}
      </Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

---

## Edge cases

- **Feature 1**: Si dos items tienen el mismo `productName`, el Command filtra ambos — correcto.
  Si el display string contiene caracteres especiales (tildes), el `.includes()` puede fallar
  para búsquedas sin tilde. Aceptable para MVP.
- **Feature 2 PDF**: `detalle` puede ser largo. `genpdf` hace wrap automático. OK.
- **Feature 2 nombre**: `LEFT JOIN users` — si el usuario fue eliminado del sistema, `created_by_name` será `NULL`. El `by: r.createdByName ?? null` muestra `—` en la tabla (comportamiento correcto).
- **Feature 3 qty package items**: si `unitPrice` es `null` para algún item de paquete, la
  línea 3 del save (`item.unitPrice != null`) silencia ese item — la cantidad NO se actualiza
  pero el item tampoco se duplica. Es el comportamiento más seguro.
- **Feature 3**: `replaceInventoryItems` borra TODOS los non-PACKAGE items y los re-inserta.
  Si hay items de otras fuentes con concept≠PACKAGE y concept≠OTHER, también serán borrados.
  Verificar que en el contexto actual solo existen concept=OTHER items como non-PACKAGE items.
