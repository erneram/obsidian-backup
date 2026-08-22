VERDICT: SHIP

## Razonamiento

Backend correcto (Option<f64> + SQL cast), merge/sort/Ver correctos, PDF sin leak. HistoryRow flat type: aceptar. Sin bloqueantes.

## Hallazgos

### Backend — `Option<f64>` mapping
🟢 BAJO: `quantity::float8 AS quantity` en SQL → `Option<f64>` en struct. sqlx mapea NULL→None automáticamente en `FromRow`. `PkgItemRow.quantity: f64` (línea 31) es un struct distinto — no confundir. `cargo build` limpio ✅

### HistoryRow flat type vs discriminated union
🟡 MEDIO (aceptar): El flat type pierde el narrowing de TypeScript — `row.number` queda `string | null` incluso dentro de `if (row.kind === 'receipt')`. Sin impacto funcional: la plantilla guarda con ternarios (`row.kind === 'receipt' ? row.number : '—'`). Para un componente interno es aceptable; en API pública exigiría NEEDS_WORK. No bloquear.

### Merge + sort + botón Ver
🟢 BAJO: `localeCompare` sobre fechas ISO (`created_at::text`) ordena correctamente — strings ISO son lexicográficamente ordenables ✅
🟢 BAJO: `v-if="row.kind === 'receipt'"` en botón Ver — solo talonarios ✅
🟢 BAJO: Movimientos con `PAYMENT`/`VOID` muestran monto en verde; el resto sin color. La columna "Tipo" (badges PAYMENT/CHARGE/VOID/ADJUSTMENT del diseño anterior) no existe en la nueva tabla — la spec no la menciona, no es regresión ✅

### Layout order
🟡 BAJO: La Tarea 1 define "Extras de Enfermería en posición 3". La Tarea 2 inserta "Paquetes Aplicados" entre Cargos y Extras, moviendo Extras a posición 4. Tensión interna entre las dos tareas — la resolución del coder es correcta dado que Tarea 2 dice explícitamente "justo debajo de la card de extras [CARGOS]". Orden final: Cargos → Paquetes Aplicados → Extras → Historial.

### PDF
🟢 BAJO: `URL.createObjectURL` + `a.click()` + `URL.revokeObjectURL` + `pdfLoading` en `finally` — sin memory leak, sin race condition ✅

### Limpieza
🟢 BAJO: `MOVEMENT_LABELS`, `movementSign`, `movementAmountClass`, `movementTypeClass` — 0 referencias en archivo ✅
