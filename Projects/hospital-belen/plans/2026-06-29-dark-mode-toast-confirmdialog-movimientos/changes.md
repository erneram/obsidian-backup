# Changes — hospital-belen-web — 4 issues frontend

**Implementado:** 2026-06-26  
**Modo:** stage-2

---

## Archivos modificados / creados

### Issue 1 — Dark mode tables

Patrón aplicado en todos los archivos: `bg-gray-50`→`bg-muted/50`, `hover:bg-gray-50`→`hover:bg-muted/30`, `text-gray-500/600`→`text-muted-foreground`, `border-gray-200`→`border-border`, badges de color → añadido `dark:bg-*/dark:text-*`.

| Archivo | Cambios |
|---|---|
| `patients/incomplete/pages/IncompletePatientListPage.vue` | bg-gray-50, hover, text-gray-500/600, border-gray-200 |
| `appointments/pages/AppointmentListPage.vue` | ídem |
| `labs/pages/LabResultListPage.vue` | + dark: en badge bg-blue-100 |
| `admin/pages/UserListPage.vue` | `bg-gray-100 text-gray-600` → `bg-muted text-muted-foreground` en statusClass |
| `admin/pages/AuditLogPage.vue` | dark: en actionClass map (CREATE/UPDATE/DELETE/LOGIN/LOGOUT) |
| `dashboard/pages/DashboardPage.vue` | border-gray-200, text-gray-500 |
| `inventory/pages/WarehouseDetailPage.vue` | dark: en badges green/gray/blue |
| `inventory/pages/ProductListPage.vue` | dark: en badges green/gray |
| `inventory/pages/InventoryMovementPage.vue` | bg-gray-50, hover, text-gray-500/600, dark: en movementBadgeClass |
| `inventory/pages/WarehouseListPage.vue` | dark: en badge green/gray |
| `admissions/pages/AdmissionDetailPage.vue` | dark: en receipt badge; movementTypeClass, statusClass (PARTIALLY_PAID/PAID/CLOSED) |
| `packages/pages/PackageDetailPage.vue` | bg-gray-50, hover, text-gray-500/600, border-gray-200, dark: en badge blue |
| `packages/pages/PackageListPage.vue` | ídem + dark: en typeBadgeClass map |

### Issue 2 — Movement amounts sign

| Archivo | Cambio |
|---|---|
| `admissions/pages/AdmissionDetailPage.vue` | Añade `movementSign()` y `movementAmountClass()`; columna Monto muestra `±Q X.XX` con color verde/azul |

### Issue 3 — Sonner dark mode

| Archivo | Cambio |
|---|---|
| `src/assets/main.css` | Añade bloque `[data-sonner-toaster][data-theme=dark]` con 15 custom properties + shadow |

### Issue 4 — ConfirmDialog (13 confirm() reemplazados)

| Archivo | Creado / Modificado |
|---|---|
| `src/components/common/ConfirmDialog.vue` | **Creado** — wrapper de AppModal |
| `admissions/pages/AdmissionDetailPage.vue` | 4 confirm() → `confirmDialog` reactive compartido |
| `patients/pages/PatientDetailPage.vue` | 1 confirm() → `confirmDeleteOpen` |
| `labs/pages/LabResultListPage.vue` | 1 confirm() → `deleteConfirmOpen + deletePendingId` |
| `admin/pages/MenuItemListPage.vue` | 1 confirm() → `deleteConfirmOpen + deletePendingId` |
| `admin/pages/RoleListPage.vue` | 1 confirm() → `deleteConfirmOpen + deletePendingRole` |
| `admin/pages/TenantListPage.vue` | 1 confirm() → `toggleConfirmOpen + toggleMessage` |
| `admin/pages/EndpointListPage.vue` | 1 confirm() → `deleteConfirmOpen + deletePendingId` |
| `admin/pages/PermissionsPage.vue` | 1 confirm() → `deleteConfirmOpen + deletePendingId` |
| `doctors/pages/DoctorDetailPage.vue` | 1 confirm() → `disconnectConfirmOpen` |
| `packages/pages/PackageDetailPage.vue` | 1 confirm() → `removeItemConfirmOpen + removePendingId` |

---

## Qué revisar puntualmente (Tester)

### Issue 1
- Activar dark mode: tablas deben tener fondos distinguibles, hover visible, badges legibles
- Verificar: IncompletePatientList, AppointmentList, LabResults, UserList, AuditLog, Dashboard cards, WarehouseDetail, ProductList, InventoryMovement, WarehouseList, AdmissionDetail, PackageDetail, PackageList

### Issue 2
- En AdmissionDetail → tab estado de cuenta → columna Monto: PAYMENT/VOID deben mostrar `-Q X.XX` en verde, CHARGE/ADJUSTMENT `+Q X.XX` en azul

### Issue 3
- Dark mode: disparar `toast.success()`, `toast.error()`, `toast.info()`, `toast()` — todos con fondo opaco visible, no transparentes

### Issue 4
- En cada pantalla listada, disparar la acción de eliminar/cerrar/desconectar → debe abrir modal ConfirmDialog (no alert nativo)
- Cancelar: modal cierra, acción no se ejecuta
- Confirmar: acción se ejecuta correctamente
- En AdmissionDetail: probar los 4 confirms (eliminar extra, eliminar cargo, cerrar admisión, eliminar admisión)
