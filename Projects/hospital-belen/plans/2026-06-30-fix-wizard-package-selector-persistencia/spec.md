# Spec — Estado de Cuenta Wizard: Package Selector UX Fix
**project:** hospital-belen  
**dispatch:** stage-1 (2026-06-30)

---

## Root cause

| Bug | Location | Cause |
|-----|----------|-------|
| Multiple packages | `StepItems.vue:307` | `push` instead of replace-all in `onSelectPackage` |
| Package disappears on back | `StepReview.vue:88-91` | Auto-applies packages on mount **and** clears `pendingPackages` → going back sees empty pending state |
| Expanded view not shown | — | Consequence of Bug 2 (no entry in `pendingPackages`) |

---

## Changes

### 1. `StepItems.vue` — single package + show selected name

**Line 307**: replace `push` with replace-all
```diff
- else ec.pendingPackages.value.push(entry)
+ else ec.pendingPackages.value.splice(0, Infinity, entry)
```

**Line 15**: show selected package name in combobox trigger
```diff
- {{ pkgLoading ? 'Cargando...' : 'Buscar paquete...' }}
+ {{ pkgLoading ? 'Cargando...' : (ec.pendingPackages.value[0]?.packageName ?? 'Buscar paquete...') }}
```

---

### 2. `StepReview.vue` — move package apply to explicit button

**Remove from `onMounted`** (lines 88-91):
```diff
- for (const pkg of ec.pendingPackages.value) {
-   await ec.applyPackage(pkg.packageId, pkg.items.map(i => ({ id: i.id, quantity: i.quantity })))
- }
- ec.pendingPackages.value = []
```

`onMounted` becomes:
```ts
onMounted(async () => {
  if (ec.pendingItems.value.length > 0) {
    await ec.confirmAndPersist()
  } else {
    await ec.refresh()
  }
})
```

**Add `applying` ref and `applyPending()` function** in `<script setup>`:
```ts
import { computed, onMounted, ref } from 'vue'
// ...
const applying = ref(false)

async function applyPending() {
  applying.value = true
  try {
    for (const pkg of ec.pendingPackages.value) {
      await ec.applyPackage(pkg.packageId, pkg.items.map(i => ({ id: i.id, quantity: i.quantity })))
    }
    ec.pendingPackages.value = []
  } finally {
    applying.value = false
  }
}
```

**Add pending-package card in template** (insert before the existing `<Card>` with items, inside `<div class="space-y-6">`):
```html
<Card v-if="ec.pendingPackages.value.length">
  <CardHeader>
    <CardTitle class="text-base">Paquete por aplicar</CardTitle>
  </CardHeader>
  <CardContent class="space-y-3">
    <div v-for="pkg in ec.pendingPackages.value" :key="pkg.packageId" class="text-sm">
      <p class="font-medium">{{ pkg.packageName }}</p>
      <div class="divide-y border rounded-md mt-1">
        <div
          v-for="item in pkg.items"
          :key="item.id"
          class="flex justify-between px-3 py-1.5"
        >
          <span class="text-muted-foreground">{{ item.productName }}</span>
          <span class="font-mono">{{ item.quantity }} × Q{{ item.unitPrice.toFixed(2) }}</span>
        </div>
      </div>
    </div>
    <Button @click="applyPending" :disabled="applying || ec.loading.value">
      {{ applying ? 'Aplicando...' : 'Aplicar paquete' }}
    </Button>
  </CardContent>
</Card>
```

---

## Behavior after fix

1. Selecting a different package replaces the previous one (splice replaces entire array).
2. Navigating items → review → back: `pendingPackages` still has the entry (never cleared until user explicitly clicks "Aplicar paquete").
3. Expanded view with `[- N +]` controls always shows when `pendingPackages.length > 0`.
4. Trigger button shows selected package name instead of static "Buscar paquete...".
5. In review, pending package is shown with full item breakdown + explicit "Aplicar paquete" button.

---

## Edge cases

- If user clicks "Aplicar paquete" in review, then goes back to items, `pendingPackages` is empty and the package appears in `packageItems` (server-applied PACKAGE items). This is correct — user intentionally confirmed it.
- If `applyPackage` fails, `pendingPackages` is NOT cleared (try/finally) — user can retry.
- `pendingItems` behavior is unchanged: still auto-persisted via `confirmAndPersist()` on review mount.

---

## Files to modify

- `hospital-belen-web/kairosaid/src/modules/estado-cuenta/components/StepItems.vue` — lines 15, 307
- `hospital-belen-web/kairosaid/src/modules/estado-cuenta/components/StepReview.vue` — onMounted, script, template
