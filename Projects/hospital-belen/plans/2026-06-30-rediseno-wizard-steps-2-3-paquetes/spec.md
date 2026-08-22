# Spec — Estado de Cuenta Wizard: UX Rediseño Steps 2 y 3
**project:** hospital-belen  
**dispatch:** stage-1 (2026-06-30)

---

## Contexto

Continúa sobre el análisis de DISPATCH anterior (package selector UX). Este DISPATCH redefine el comportamiento de Steps 2 y 3 y dónde se ejecuta `applyPackage`.

---

## Root cause / estado actual

- `StepItems.vue`: ícono de basura en header del paquete elimina el paquete entero; no hay per-item toggle
- `StepReview.vue`: tiene botones Excel y PDF; el DISPATCH 3 anterior specceó "Aplicar paquete" (no implementado aún → ignorar, no agregar)
- `EstadoCuentaWizardPage.vue`: `onFinish()` solo reset + navigate; no aplica packages

---

## Cambios

### 1. `types/index.ts`

Agregar `excluded` a `PendingPackageItem`:

```diff
 export type PendingPackageItem = {
   id: string
   productName: string
   unitPrice: number
   quantity: number
+  excluded?: boolean
 }
```

---

### 2. `StepItems.vue`

#### 2a. Toggle strikethrough por ítem (replace current Trash2 behavior in item rows)

El ítem de paquete NO se elimina — se tacha (toggle). El Trash2 en el **header** del paquete sigue eliminando el paquete completo (sin cambio).

Agregar función toggle en `<script setup>`:
```ts
function toggleExcludeItem(pkgIdx: number, itemIdx: number) {
  const item = ec.pendingPackages.value[pkgIdx]?.items[itemIdx]
  if (item) item.excluded = !item.excluded
}
```

#### 2b. Template — fila de ítem del paquete

Reemplazar el bloque `<div v-for="(item, itemIdx) in pkg.items"` (dentro del paquete pendiente) por:

```html
<div
  v-for="(item, itemIdx) in pkg.items"
  :key="item.id"
  class="flex items-center gap-2 px-3 py-2"
  :class="{ 'opacity-50': item.excluded }"
>
  <span class="flex-1 truncate text-sm" :class="{ 'line-through': item.excluded }">
    {{ item.productName }}
  </span>
  <span class="text-muted-foreground w-20 text-right font-mono text-sm">
    Q{{ item.unitPrice.toFixed(2) }}
  </span>
  <div class="flex items-center gap-1">
    <Button type="button" variant="ghost" size="icon" class="h-7 w-7"
      :disabled="item.quantity <= 1 || item.excluded"
      @click="adjustPendingItemQty(pkgIdx, itemIdx, -1)"
    ><Minus class="h-3 w-3" /></Button>
    <span class="w-8 text-center tabular-nums text-sm">{{ item.quantity }}</span>
    <Button type="button" variant="ghost" size="icon" class="h-7 w-7"
      :disabled="item.excluded"
      @click="adjustPendingItemQty(pkgIdx, itemIdx, 1)"
    ><Plus class="h-3 w-3" /></Button>
  </div>
  <span class="w-20 text-right font-mono text-sm">
    Q{{ (item.excluded ? 0 : item.quantity * item.unitPrice).toFixed(2) }}
  </span>
  <Button type="button" variant="ghost" size="icon"
    class="h-7 w-7"
    :class="item.excluded ? 'text-muted-foreground' : 'text-destructive hover:text-destructive'"
    @click="toggleExcludeItem(pkgIdx, itemIdx)"
  ><Trash2 class="h-3.5 w-3.5" /></Button>
</div>
```

#### 2c. Trigger del combobox — mostrar nombre del paquete seleccionado (de DISPATCH anterior)

```diff
- {{ pkgLoading ? 'Cargando...' : 'Buscar paquete...' }}
+ {{ pkgLoading ? 'Cargando...' : (ec.pendingPackages.value[0]?.packageName ?? 'Buscar paquete...') }}
```

#### 2d. Single-package enforce (de DISPATCH anterior)

```diff
- else ec.pendingPackages.value.push(entry)
+ else ec.pendingPackages.value.splice(0, Infinity, entry)
```

---

### 3. `StepReview.vue`

#### 3a. Eliminar botones Excel y PDF

Eliminar el bloque completo:
```html
<div class="flex justify-end gap-2">
  <Button type="button" variant="outline" ...>Exportar Excel</Button>
  <Button type="button" ...>Generar PDF</Button>
</div>
```
(y los imports de `FileSpreadsheet`, `FileDown` de lucide si quedan sin usar)

#### 3b. Card informativa del paquete pendiente (sin botón de acción)

Insertar antes de la card principal de items. Muestra los ítems con tachado si `excluded`:

```html
<Card v-if="ec.pendingPackages.value.length">
  <CardHeader>
    <CardTitle class="text-base">Paquete por aplicar</CardTitle>
  </CardHeader>
  <CardContent>
    <div v-for="pkg in ec.pendingPackages.value" :key="pkg.packageId" class="text-sm">
      <p class="font-medium mb-1">{{ pkg.packageName }}</p>
      <div class="divide-y border rounded-md">
        <div
          v-for="item in pkg.items"
          :key="item.id"
          class="flex justify-between px-3 py-1.5"
          :class="{ 'opacity-50': item.excluded }"
        >
          <span :class="{ 'line-through': item.excluded }">{{ item.productName }}</span>
          <span class="font-mono">
            {{ item.excluded ? '—' : `${item.quantity} × Q${item.unitPrice.toFixed(2)}` }}
          </span>
        </div>
      </div>
    </div>
    <p class="text-xs text-muted-foreground mt-2">
      Se aplicará al presionar "VER".
    </p>
  </CardContent>
</Card>
```

#### 3c. Totales — sumar paquete pendiente si aún no está en servidor

Agregar computed en script:
```ts
const pendingPackageTotal = computed(() =>
  ec.pendingPackages.value.reduce(
    (sum, pkg) => sum + pkg.items
      .filter(i => !i.excluded)
      .reduce((s, i) => s + i.quantity * i.unitPrice, 0),
    0
  )
)
```

Actualizar la fila de total paquete en template:
```diff
- <span class="font-mono">Q {{ totalPackage.toFixed(2) }}</span>
+ <span class="font-mono">Q {{ (totalPackage + pendingPackageTotal).toFixed(2) }}</span>
```

#### 3d. `onMounted` — NO aplica paquetes (ya definido en DISPATCH anterior)

```ts
onMounted(async () => {
  if (ec.pendingItems.value.length > 0) {
    await ec.confirmAndPersist()
  } else {
    await ec.refresh()
  }
})
```
Sin loop de `applyPackage`, sin `ec.pendingPackages.value = []`.

---

### 4. `EstadoCuentaWizardPage.vue`

#### 4a. `onFinish()` aplica paquetes pendientes antes de navegar

```diff
-function onFinish() {
+async function onFinish() {
   const id = ec.admissionId.value
+  for (const pkg of ec.pendingPackages.value) {
+    await ec.applyPackage(
+      pkg.packageId,
+      pkg.items.filter(i => !i.excluded).map(i => ({ id: i.id, quantity: i.quantity }))
+    )
+  }
   ec.reset()
   router.push(id ? `/admissions/${id}` : '/admissions')
 }
```

#### 4b. Deshabilitar botón "VER" mientras se aplica

```diff
- <Button v-else variant="outline" @click="onFinish">
+ <Button v-else variant="outline" :disabled="ec.loading.value" @click="onFinish">
```

---

## Comportamiento resultante

1. En Step 2, el Trash2 por ítem **tacha** el ítem (toggle) — los controles `[- N +]` se deshabilitan para ítems tachados; total de ese ítem muestra Q0.00.
2. El Trash2 en el header del paquete sigue eliminando el paquete completo (sin cambio).
3. En Step 3, card informativa muestra el paquete con ítems tachados visibles. No hay botones de acción.
4. "VER" aplica paquetes (excluyendo ítems tachados), luego reset + navigate.
5. El usuario puede regresar de Step 3 a Step 2, ajustar cantidades o togglear ítems, volver a Step 3 — `pendingPackages` persiste (no se limpia en Step 3).

---

## Edge cases

- Si todos los ítems de un paquete están tachados, `applyPackage` se llama con `items: []` — el coder debe verificar que el endpoint acepta array vacío (o skipear el apply en ese caso).
- `excluded` se inicializa en `undefined` (falsy) al crear la entry en `onSelectPackage` — no necesita cambio en la lógica de creación.
- Si `pendingPackages` está vacío al presionar "VER", el loop no hace nada — behavior correcto.

---

## Archivos a modificar

- `hospital-belen-web/kairosaid/src/modules/estado-cuenta/types/index.ts`
- `hospital-belen-web/kairosaid/src/modules/estado-cuenta/components/StepItems.vue`
- `hospital-belen-web/kairosaid/src/modules/estado-cuenta/components/StepReview.vue`
- `hospital-belen-web/kairosaid/src/modules/estado-cuenta/pages/EstadoCuentaWizardPage.vue`
