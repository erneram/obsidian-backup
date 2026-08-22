# Spec — hospital-belen-web — 4 issues frontend

**Generado:** 2026-06-26  
**Modo:** stage-1

---

## Issue 1 — Dark/light mode en tablas

### Diagnóstico

Las tablas usan clases Tailwind con colores hardcoded que no adaptan al tema:
- `bg-gray-50`, `bg-gray-100` → en dark mode se ven oscuro-sobre-oscuro (imperceptibles o en contraste mal)
- `hover:bg-gray-50` → gris claro en hover no se ve en dark
- `text-gray-500`, `text-gray-600` → pueden confundirse con el fondo en dark
- Badges de estado: `bg-green-100 text-green-800`, `bg-red-100 text-red-700` etc. → saturados en light, casi invisibles en dark

Algunos archivos ya tienen el patrón correcto (ej. `AdmissionListPage.vue:771` tiene `dark:bg-blue-900/30 dark:text-blue-400`). Hay que extenderlo a todos.

### Patrón de reemplazo

| Clase actual | Clase correcta |
|---|---|
| `bg-gray-50` | `bg-muted/50` |
| `bg-gray-100` | `bg-muted` |
| `hover:bg-gray-50` | `hover:bg-muted/30` |
| `hover:bg-gray-100` | `hover:bg-muted/40` |
| `text-gray-500` | `text-muted-foreground` |
| `text-gray-600` | `text-muted-foreground` |
| `border-gray-200` | `border-border` |
| `bg-green-100 text-green-800` | `bg-green-100 text-green-800 dark:bg-green-900/30 dark:text-green-400` |
| `bg-red-100 text-red-800` | `bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-400` |
| `bg-red-100 text-red-700` | `bg-red-100 text-red-700 dark:bg-red-900/30 dark:text-red-400` |
| `bg-blue-100 text-blue-800` | `bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-400` |
| `bg-yellow-100 text-yellow-800` | `bg-yellow-100 text-yellow-800 dark:bg-yellow-900/20 dark:text-yellow-400` |

### Archivos a modificar (SOLO tablas — no PublicBookingPage, LoginPage, ReceiptPreview que son páginas no temadas)

Prioridad alta — usan HTML nativo (`thead`/`tr`) sin los componentes Table:
- `src/modules/patients/incomplete/pages/IncompletePatientListPage.vue` — `bg-gray-50`, `hover:bg-gray-50`
- `src/modules/appointments/pages/AppointmentListPage.vue` — `bg-gray-50`, `hover:bg-gray-50`

Prioridad alta — usan componentes Table pero con colores hardcoded:
- `src/modules/labs/pages/LabResultListPage.vue` — `<TableHeader class="bg-gray-50">`, `hover:bg-gray-50`, `bg-blue-100 text-blue-800`
- `src/modules/admin/pages/UserListPage.vue` — `bg-gray-50`
- `src/modules/admin/pages/AuditLogPage.vue` — `bg-gray-50`
- `src/modules/dashboard/pages/DashboardPage.vue` — `bg-gray-50`
- `src/modules/inventory/pages/WarehouseDetailPage.vue` — `bg-gray-50`
- `src/modules/inventory/pages/ProductListPage.vue` — `bg-gray-50`
- `src/modules/inventory/pages/InventoryMovementPage.vue` — `bg-gray-50`
- `src/modules/inventory/pages/WarehouseListPage.vue` — `bg-gray-50` (en cards si existe)
- `src/modules/admissions/pages/AdmissionDetailPage.vue` — status badges sin dark: para PARTIALLY_PAID/PAID/CLOSED (solo OPEN tiene dark:)
- `src/modules/packages/pages/PackageDetailPage.vue` — `bg-gray-50`, status badges
- `src/modules/packages/pages/PackageListPage.vue` — `bg-gray-50`, status badges

Regla: nunca tocar `PublicBookingPage.vue`, `LoginPage.vue`, `ReceiptPreview.vue` — son páginas de print/public que no requieren dark mode.

### Verificación

Cambiar el modo a dark en el browser y comprobar:
1. Fondos de `<TableHeader>` y `<thead>` se ven distinguibles del cuerpo
2. Hover sobre filas tiene contraste visible
3. Badges de estado son legibles (texto claro sobre fondo oscuro)

---

## Issue 2 — Estado de cuenta: selector de cantidad por ítem de paquete

**Clarificación del humano (2026-06-26):** Al seleccionar un paquete en el estado de cuenta, cada línea de ítem debe mostrar:

```
Nombre del ítem  |  Q(precio unitario)  |  [- N +]  |  Q(N × precio unitario)
```

Donde `N` es la cantidad por defecto (`default_quantity`) que trae el ítem del paquete. El usuario puede ajustar N con los botones `-` y `+`, y el total se recalcula automáticamente.

### Comportamiento esperado
- Al agregar un paquete al estado de cuenta, cada ítem aparece con su `default_quantity` como valor inicial de N.
- Botón `-`: decrementa N (mínimo 1, o 0 si el ítem es opcional).
- Botón `+`: incrementa N.
- El precio total del ítem = N × precio_unitario (se actualiza en tiempo real).
- El total del estado de cuenta suma todos los ítems con sus N actuales.

### Archivos a modificar

| Archivo | Cambio |
|---|---|
| `src/modules/admissions/pages/AdmissionDetailPage.vue` | Sección de ítems de paquete en estado de cuenta |

### Cambio exacto (basado en interpretación A)

**Función helper** en el `<script setup>`:
```ts
function movementSign(type: string): string {
  return ['PAYMENT', 'VOID'].includes(type) ? '-' : '+'
}
function movementAmountClass(type: string): string {
  return ['PAYMENT', 'VOID'].includes(type) ? 'text-green-600 dark:text-green-400' : 'text-blue-600 dark:text-blue-400'
}
```

**Template** (en el `<TableCell>` de la columna Monto, actualmente línea ~426):
```html
<!-- antes: Q {{ formatAmount(m.amount) }} -->
<span :class="movementAmountClass(m.movementType)" class="font-mono">
  {{ movementSign(m.movementType) }}Q {{ formatAmount(m.amount) }}
</span>
```

### Edge cases
- Si el coder confirma que la interpretación correcta es B (ítems de cuenta con `+`/`-`), cambiar en `StepReview.vue` y en la lista de ítems de `AdmissionDetailPage.vue` para que `ADVANCE`/`PAYMENT` concepts muestren `-Q` y el resto `+Q`
- `formatAmount` sigue siendo positivo (el signo se agrega por separado)

---

## Issue 3 — Toast/Sonner transparente en dark mode

### Diagnóstico

`src/assets/main.css` tiene overrides para `[data-sonner-toaster][data-theme=light]` pero **no para `[data-sonner-toaster][data-theme=dark]`**.

vue-sonner en dark mode aplica sus propias CSS custom properties con valores por defecto oscuros, pero sobre el background del app (`--background: 0 0% 7%` en dark) el contraste es insuficiente — los toasts se ven casi transparentes/invisibles.

### Archivos a modificar

| Archivo | Cambio |
|---|---|
| `src/assets/main.css` | Añadir bloque dark al final |

### Bloque a agregar (al final de `main.css`, después del bloque light existente)

```css
/* ─── Sonner / vue-sonner toast overrides (dark theme) ──────────────────────
   Espejo del bloque light: misma especificidad 0,2,0 con !important para
   ganar la cascada del CSS inyectado por vue-sonner en runtime.
   ──────────────────────────────────────────────────────────────────────── */
[data-sonner-toaster][data-theme=dark] {
  --normal-bg: hsl(0 0% 14%) !important;
  --normal-border: hsl(0 0% 22%) !important;
  --normal-text: hsl(0 0% 95%) !important;
  --success-bg: hsl(140 35% 16%) !important;
  --success-border: hsl(140 40% 28%) !important;
  --success-text: hsl(143 60% 70%) !important;
  --info-bg: hsl(210 35% 16%) !important;
  --info-border: hsl(210 40% 30%) !important;
  --info-text: hsl(208 80% 72%) !important;
  --warning-bg: hsl(38 35% 16%) !important;
  --warning-border: hsl(40 40% 30%) !important;
  --warning-text: hsl(45 90% 68%) !important;
  --error-bg: hsl(0 35% 16%) !important;
  --error-border: hsl(0 40% 30%) !important;
  --error-text: hsl(0 85% 72%) !important;
}

[data-sonner-toaster][data-theme=dark] [data-sonner-toast][data-styled=true] {
  box-shadow: 0 8px 24px -4px rgba(0, 0, 0, 0.5), 0 4px 8px -2px rgba(0, 0, 0, 0.3);
}
```

### Edge cases
- `RouterLayoutView.vue` usa `<Sonner rich-colors ...>` — `rich-colors` hace que vue-sonner aplique estilos por tipo (success/error etc.). Con `rich-colors`, las variables `--success-bg` etc. toman efecto; sin él, solo `--normal-bg`.
- Verificar en dark mode con `toast.success()`, `toast.error()`, `toast.info()`, `toast()` (normal)

---

## Issue 4 — Modal de confirmación (cerrar admisión + eliminar todos los `confirm()`)

### Diagnóstico

Hay **13 lugares** con `confirm()` nativo del navegador en el código. La tarea pide:
1. Reemplazar TODOS los `confirm()` con modales del design system
2. Prioridad: el de cerrar admisión (`AdmissionDetailPage.vue:896`) debe tener un modal de "¿Estás seguro?"

### Componente reutilizable (crear)

| Archivo | Acción |
|---|---|
| `src/components/common/ConfirmDialog.vue` | **Crear** |

```vue
<!-- ConfirmDialog.vue — ponytail: wrapper fino sobre AppModal para confirm() -->
<template>
  <AppModal
    :open="open"
    :title="title"
    :description="message"
    show-cancel
    show-confirm
    :cancel-label="cancelLabel"
    :confirm-label="confirmLabel"
    :confirm-variant="variant"
    :loading="loading"
    @update:open="$emit('update:open', $event)"
    @confirm="$emit('confirm')"
    @cancel="$emit('update:open', false)"
  />
</template>

<script setup lang="ts">
import AppModal from './AppModal.vue'

withDefaults(defineProps<{
  open: boolean
  title?: string
  message?: string
  confirmLabel?: string
  cancelLabel?: string
  variant?: 'default' | 'destructive'
  loading?: boolean
}>(), {
  title: '¿Estás seguro?',
  confirmLabel: 'Confirmar',
  cancelLabel: 'Cancelar',
  variant: 'default',
  loading: false,
})

defineEmits<{ 'update:open': [boolean]; confirm: [] }>()
</script>
```

### Patrón de reemplazo por archivo

Para cada `confirm()` se sigue este patrón:

**En script:**
```ts
const confirmXxxOpen = ref(false)
// Donde antes estaba: if (!confirm('...')) return; await doAction()
function requestXxx() { confirmXxxOpen.value = true }
async function handleXxxConfirmed() {
  confirmXxxOpen.value = false
  await doAction()
}
```

**En template:**
```html
<ConfirmDialog
  v-model:open="confirmXxxOpen"
  title="Título"
  message="Mensaje descriptivo."
  confirm-label="Confirmar"
  variant="destructive"
  @confirm="handleXxxConfirmed"
/>
```

### Inventario completo de `confirm()` a reemplazar

| # | Archivo | Línea | Mensaje actual | variant |
|---|---|---|---|---|
| 1 | `admissions/AdmissionDetailPage.vue` | 896 | ¿Cerrar esta admisión? No se podrán realizar más cambios. | `destructive` |
| 2 | `admissions/AdmissionDetailPage.vue` | 901 | ¿Eliminar esta admisión? Se marcará como cancelada. | `destructive` |
| 3 | `admissions/AdmissionDetailPage.vue` | 724 | ¿Eliminar este extra? | `destructive` |
| 4 | `admissions/AdmissionDetailPage.vue` | 809 | ¿Eliminar este cargo? | `destructive` |
| 5 | `patients/PatientDetailPage.vue` | 532 | ¿Eliminar al paciente ${fullName}? Esta acción lo marcará como inactivo. | `destructive` |
| 6 | `labs/LabResultListPage.vue` | 38 | t('labs.confirmDelete') | `destructive` |
| 7 | `admin/MenuItemListPage.vue` | 217 | ¿Eliminar este ítem del menú? | `destructive` |
| 8 | `admin/RoleListPage.vue` | 328 | ¿Eliminar el rol "${role.name}"? | `destructive` |
| 9 | `admin/TenantListPage.vue` | 143 | ¿{action} la clínica "${t.name}"? | `default` |
| 10 | `admin/EndpointListPage.vue` | 275 | ¿Eliminar este endpoint? | `destructive` |
| 11 | `admin/PermissionsPage.vue` | 195 | ¿Eliminar este permiso? | `destructive` |
| 12 | `doctors/DoctorDetailPage.vue` | 126 | ¿Desconectar Google Calendar? | `default` |
| 13 | `packages/PackageDetailPage.vue` | 104 | t('packages.item.confirmRemove') | `destructive` |

### Nota para AdmissionDetailPage (admisiones múltiples en mismo archivo)

`AdmissionDetailPage.vue` tiene 4 confirm() en el mismo archivo. El coder puede usar un solo `ConfirmDialog` genérico con `v-model:open`, `title`, `message` y `@confirm` enlazados a refs:

```ts
const confirmDialog = reactive({
  open: false,
  title: '',
  message: '',
  variant: 'destructive' as 'default' | 'destructive',
  onConfirm: () => {},
})

function openConfirm(title: string, message: string, variant: 'default'|'destructive', action: () => void) {
  confirmDialog.title = title
  confirmDialog.message = message
  confirmDialog.variant = variant
  confirmDialog.onConfirm = action
  confirmDialog.open = true
}
```

Esto evita tener 4 refs de diálogo separados en el mismo componente.

### Edge cases
- `TenantListPage.vue:143` usa un mensaje dinámico con el `action` (activate/deactivate) — el `ConfirmDialog` debe recibir el mensaje ya interpolado antes de abrir
- `LabResultListPage.vue:38` usa i18n `t('labs.confirmDelete')` — pasar como `:message="t('labs.confirmDelete')"`
- `RoleListPage.vue:328` interpola el nombre del rol — igual, computar el string antes de pasar

---

## Orden de implementación sugerido

1. `src/components/common/ConfirmDialog.vue` — crear el componente (base para Issue 4)
2. `src/assets/main.css` — agregar bloque dark de Sonner (Issue 3, 3 líneas)
3. Dark mode tables — todos los archivos de Issue 1 (reemplazos mecánicos, en bloque)
4. `AdmissionDetailPage.vue` — confirm() → ConfirmDialog + movimiento sign (Issues 2 y 4 combinados)
5. Resto de archivos con confirm() — Issues 4

---

## Patrones existentes a seguir

- `src/components/common/AppModal.vue` — base del ConfirmDialog
- `AdmissionListPage.vue:771` — patrón dark: correcto para badges de estado
- `src/assets/main.css:156` — patrón `[data-sonner-toaster][data-theme=light]` a replicar para dark
