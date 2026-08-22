# Spec — AdmissionDetailPage: Layout + Columnas + PDF
**project:** hospital-belen  
**dispatch:** stage-1 (2026-07-03)

---

## Contexto

Este DISPATCH continúa el rediseño de `AdmissionDetailPage.vue` (ver DISPATCH anterior donde CARGOS pasó a mostrar `extras` y EXTRAS DE ENFERMERÍA usa `addItem` conceptual).

---

## Tarea 1 — Reestructura de layout

### Orden actual de secciones (de arriba a abajo)
1. Datos Clínicos + Balance de Cuenta (grid 2 col)
2. Cargos de Inventario (extras)
3. Talonarios (receiptService)
4. Extras de Enfermería (conceptualItems de detail.items)
5. Add Extra Dialog
6. Historial de Pagos / movements

### Orden objetivo
1. Datos Clínicos + Balance de Cuenta — sin cambios ✓
2. Cargos de Inventario — sin cambios ✓
3. **Extras de Enfermería** — mover aquí (subir desde posición 4 a 3)
4. **Historial de Pagos** — sección unificada (Talonarios + movements)

---

### Sección "Historial de Pagos" unificada

Reemplazar las dos cards separadas (Talonarios + movements) por **una sola** con:
- Header: "Historial de Pagos" + botón `<Plus /> Crear talonario` (link a `/receipts/new?admissionId=...`)
- Tabla unificada de talonarios y movimientos, ordenada por fecha

#### Lógica de merge

Construir un array `historyRows` combinando ambas fuentes con un tipo discriminador:

```ts
type HistoryRow =
  | { kind: 'receipt'; id: string; number: string; amount: number; notes: string | null; by: string | null; date: string }
  | { kind: 'movement'; id: string; movementType: string; amount: number; notes: string | null; by: string | null; date: string }

const historyRows = computed((): HistoryRow[] => {
  const receipts: HistoryRow[] = admissionReceipts.value.map(r => ({
    kind: 'receipt',
    id: r.id,
    number: r.receiptNumber,
    amount: r.totalAmount ?? 0,
    notes: r.concepto ?? null,
    by: null,
    date: r.createdAt,
  }))
  const moves: HistoryRow[] = movements.value.map(m => ({
    kind: 'movement',
    id: m.id,
    movementType: m.movementType,
    amount: m.amount,
    notes: m.notes,
    by: m.performedByName,
    date: m.createdAt,
  }))
  return [...receipts, ...moves].sort((a, b) => a.date.localeCompare(b.date))
})
```

#### Template de la tabla unificada

Columnas: N° | Monto | Notas | Por | Fecha | [Ver]

```html
<Card>
  <CardHeader class="flex flex-row items-center justify-between">
    <CardTitle class="text-base">Historial de Pagos</CardTitle>
    <Button
      v-if="detail.statement.status !== 'CLOSED'"
      size="sm"
      variant="outline"
      @click="$router.push(`/receipts/new?admissionId=${route.params.id}`)"
    >
      <Plus :size="14" class="mr-1" /> Crear talonario
    </Button>
  </CardHeader>
  <CardContent>
    <div v-if="receiptsLoading" class="py-4 text-center text-muted-foreground text-sm">Cargando...</div>
    <Table v-else class="min-w-full divide-y divide-border text-sm">
      <TableHeader>
        <TableRow>
          <TableHead class="px-4 py-2 text-left text-xs font-medium text-muted-foreground uppercase">N°</TableHead>
          <TableHead class="px-4 py-2 text-right text-xs font-medium text-muted-foreground uppercase">Monto</TableHead>
          <TableHead class="px-4 py-2 text-left text-xs font-medium text-muted-foreground uppercase">Notas</TableHead>
          <TableHead class="px-4 py-2 text-left text-xs font-medium text-muted-foreground uppercase">Por</TableHead>
          <TableHead class="px-4 py-2 text-left text-xs font-medium text-muted-foreground uppercase">Fecha</TableHead>
          <TableHead class="px-4 py-2 w-16" />
        </TableRow>
      </TableHeader>
      <TableBody class="divide-y divide-border">
        <TableRow v-if="!historyRows.length">
          <TableCell colspan="6" class="px-4 py-8 text-center text-muted-foreground">Sin registros.</TableCell>
        </TableRow>
        <TableRow v-for="row in historyRows" :key="row.id" class="hover:bg-muted/30">
          <TableCell class="px-4 py-2 font-mono text-xs text-muted-foreground">
            {{ row.kind === 'receipt' ? row.number : '—' }}
          </TableCell>
          <TableCell class="px-4 py-2 text-right font-mono">
            <span :class="row.kind === 'movement' && ['PAYMENT','VOID'].includes(row.movementType ?? '') ? 'text-green-600 dark:text-green-400' : ''">
              Q {{ formatAmount(row.amount) }}
            </span>
          </TableCell>
          <TableCell class="px-4 py-2 text-muted-foreground text-sm">{{ row.notes ?? '—' }}</TableCell>
          <TableCell class="px-4 py-2 text-muted-foreground text-sm">{{ row.by ?? '—' }}</TableCell>
          <TableCell class="px-4 py-2 text-muted-foreground text-xs">{{ formatDate(row.date) }}</TableCell>
          <TableCell class="px-4 py-2 text-right">
            <Button
              v-if="row.kind === 'receipt'"
              variant="ghost"
              size="sm"
              @click="$router.push(`/receipts/${row.id}`)"
            >Ver</Button>
          </TableCell>
        </TableRow>
      </TableBody>
    </Table>
  </CardContent>
</Card>
```

---

## Tarea 2 — Columnas en Cargos de Inventario

### Diagnóstico

`AdmissionExtra` (fuente de la tabla CARGOS) ya tiene `quantity`, `unitPrice`, `total` → columnas disponibles sin cambio backend.

Para items de **paquetes aplicados** (`concept=PACKAGE` en `detail.items`): el struct Rust `AccountStatementItem` NO incluye `quantity`/`unit_price`. La tabla DB sí tiene estas columnas.

### Cambios backend (pequeño)

**`hospital-belen-api/src/domain/admission/mod.rs`** — extender struct:
```diff
 pub struct AccountStatementItem {
     pub id: Uuid,
     pub tenant_id: Uuid,
     pub account_statement_id: Uuid,
     pub concept: String,
     pub description: String,
     pub original_amount: f64,
     pub paid_amount: f64,
+    pub quantity: Option<f64>,
+    pub unit_price: Option<f64>,
     pub created_at: String,
 }
```

**`hospital-belen-api/src/infrastructure/database/repositories/admission_repository.rs`** — actualizar la query del `get()` (líneas ~194-200):
```diff
 SELECT
     id, tenant_id, account_statement_id,
     concept::text           AS concept,
     description,
     original_amount::float8 AS original_amount,
     paid_amount::float8     AS paid_amount,
+    quantity::float8        AS quantity,
+    unit_price::float8      AS unit_price,
     created_at::text        AS created_at
 FROM account_statement_items
```

**`hospital-belen-web/kairosaid/src/modules/admissions/types/index.ts`** — actualizar interfaz:
```diff
 export interface AccountStatementItem {
   id: string
   tenantId: string
   accountStatementId: string
   concept: string
   description: string
   originalAmount: number
   paidAmount: number
+  quantity?: number
+  unitPrice?: number
   createdAt: string
 }
```

### Cambios frontend — tabla Cargos de Inventario (PACKAGE items)

La sección CARGOS muestra `extras`. El OPEN QUESTION del spec anterior sobre packages se resuelve: agregar una card de solo lectura para PACKAGE items en `detail.items`, justo debajo de la card de `extras`.

Agregar computed en `AdmissionDetailPage.vue`:
```ts
const packageItems = computed(
  () => detail.value?.items.filter(i => i.concept === 'PACKAGE') ?? []
)
```

Agregar card after the CARGOS card:
```html
<Card v-if="packageItems.length">
  <CardHeader>
    <CardTitle class="text-base">Paquetes Aplicados</CardTitle>
  </CardHeader>
  <CardContent>
    <Table class="min-w-full divide-y divide-border text-sm">
      <TableHeader>
        <TableRow>
          <TableHead class="px-4 py-2 text-left text-xs uppercase text-muted-foreground">Descripción</TableHead>
          <TableHead class="px-4 py-2 text-right text-xs uppercase text-muted-foreground">Cant.</TableHead>
          <TableHead class="px-4 py-2 text-right text-xs uppercase text-muted-foreground">Precio Unit.</TableHead>
          <TableHead class="px-4 py-2 text-right text-xs uppercase text-muted-foreground">Total</TableHead>
          <TableHead class="px-4 py-2 w-12" />
        </TableRow>
      </TableHeader>
      <TableBody class="divide-y divide-border">
        <TableRow v-for="item in packageItems" :key="item.id" class="hover:bg-muted/30">
          <TableCell class="px-4 py-2">{{ item.description }}</TableCell>
          <TableCell class="px-4 py-2 text-right font-mono">{{ item.quantity ?? '—' }}</TableCell>
          <TableCell class="px-4 py-2 text-right font-mono">
            {{ item.unitPrice != null ? `Q ${formatAmount(item.unitPrice)}` : '—' }}
          </TableCell>
          <TableCell class="px-4 py-2 text-right font-mono">Q {{ formatAmount(item.originalAmount) }}</TableCell>
          <TableCell class="px-4 py-2 text-center">
            <Button
              v-if="detail.statement.status !== 'CLOSED'"
              type="button" variant="ghost"
              class="text-destructive hover:text-destructive/70 p-1 h-auto"
              @click="confirmRemoveItem(item.id)"
            ><Trash2 :size="14" /></Button>
          </TableCell>
        </TableRow>
      </TableBody>
    </Table>
  </CardContent>
</Card>
```

La card de `extras` (CARGOS de Inventario) ya tiene: Descripción | Cant. | Precio Unit. | Total — sin cambios.

---

## Tarea 3 — Botón PDF

### Diagnóstico: endpoint existente

`GET /api/admissions/{id}/pdf` ya existe y genera un PDF con items + extras + payments. No se necesitan cambios al backend.

Frontend service existente: `estadoCuentaService.downloadStatementPdf(id)` → llama al endpoint, devuelve Blob.

### Cambios solo en frontend (AdmissionDetailPage.vue)

**Import** (agregar):
```ts
import { estadoCuentaService } from '@/modules/estado-cuenta/services/estadoCuentaService'
```

**Estado y función** (agregar):
```ts
const pdfLoading = ref(false)

async function downloadPdf() {
  pdfLoading.value = true
  try {
    const blob = await estadoCuentaService.downloadStatementPdf(route.params.id as string)
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `estado-cuenta-${detail.value?.admission.admissionNumber ?? route.params.id}.pdf`
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    URL.revokeObjectURL(url)
  } finally {
    pdfLoading.value = false
  }
}
```

**Botón en el header** (agregar en el div de botones junto a "Imprimir"):
```html
<Button variant="outline" size="sm" :disabled="pdfLoading" @click="downloadPdf">
  <FileDown :size="14" class="mr-1" /> {{ pdfLoading ? 'Generando...' : 'Descargar PDF' }}
</Button>
```

Import de `FileDown` de lucide-vue-next (probablemente ya importado; verificar).

---

## Resumen de archivos a modificar

| Archivo | Tarea | Cambio |
|---------|-------|--------|
| `hospital-belen-api/src/domain/admission/mod.rs` | 2 | Agregar `quantity`, `unit_price` a `AccountStatementItem` |
| `hospital-belen-api/src/infrastructure/database/repositories/admission_repository.rs` | 2 | Agregar campos en query `get()` |
| `hospital-belen-web/kairosaid/src/modules/admissions/types/index.ts` | 2 | Agregar `quantity?`, `unitPrice?` a `AccountStatementItem` |
| `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue` | 1, 2, 3 | Layout + packageItems card + PDF button |
