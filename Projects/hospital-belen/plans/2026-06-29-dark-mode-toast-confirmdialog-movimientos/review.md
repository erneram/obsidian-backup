VERDICT: SHIP

## Razonamiento

Los 4 issues implementados correctamente. Fix-request resuelto. Sin regresiones, sin scope leakage, sin confirm() nativos restantes.

## Hallazgos

### Issue 1 — Dark mode tables
🟢 BAJO: 0 clases `bg-gray-50/100`, `text-gray-500/600`, `border-gray-200`, `hover:bg-gray-*` en los 13 archivos objetivo. AdmissionDetailPage `statusClass` cubre OPEN/PARTIALLY_PAID/PAID/CLOSED con dark: variantes correctas.
🟢 BAJO: Fix-request resuelto — `WarehouseDetailPage.vue:93-94` (`dark:bg-green-900/30 dark:text-green-400`) y `PackageListPage.vue:212-213` (`dark:hover:bg-green-900/50`) confirmados.

### Issue 2 — Movement sign
🟢 BAJO: `movementSign()` y `movementAmountClass()` en líneas 788-793. Template líneas 427-429: PAYMENT/VOID → `-Q` verde, resto → `+Q` azul. Correcto per spec.

### Issue 3 — Sonner dark CSS
🟢 BAJO: Bloque `[data-sonner-toaster][data-theme=dark]` en `main.css:182-202` — 15 custom properties + shadow. Coincide exactamente con spec.

### Issue 4 — ConfirmDialog
🟢 BAJO: `ConfirmDialog.vue` creado — wrapper exacto al spec sobre `AppModal`. Todos sus props (`showCancel/showConfirm/cancelLabel/confirmVariant/loading`) verificados en AppModal.vue.
🟢 BAJO: 0 llamadas nativas `confirm()` restantes en todos los 10 archivos afectados. Todos tienen ConfirmDialog import + uso en template.
🟢 BAJO: AdmissionDetailPage usa `reactive confirmDialog` compartido para sus 4 casos — patrón limpio, evita 4 refs separados.

## Sin hallazgos de seguridad, commits no pedidos, ni cambios fuera de scope.
