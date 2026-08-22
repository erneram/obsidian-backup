# Fix Request — Historial de Pagos + quantity column + PDF + Imprimir

## Bug 1 — Historial de Pagos muestra 2 filas por talonario
Cada talonario T-XXXX aparece dos veces: una como `receipt` y otra como `movement` con `notes: "Talonario T-XXXX — ..."`. Cuando se crea un talonario, también se genera un movement de tipo PAYMENT que lo referencia. El merge en `historyRows` incluye ambos, duplicando la entrada.
**Fix**: filtrar del array `movements` cualquier entry cuyas `notes` contengan "Talonario T-" (o movimientos de tipo PAYMENT que tienen un talonario asociado). Solo mostrar los talonarios como source of truth para pagos. Alternativamente: si el movimiento tiene `notes` que empieza con "Talonario", omitirlo de `moves`.

## Bug 2 — Error al crear Extras de Enfermería: `no column found for name: quantity`
Al agregar `quantity::float8 AS quantity` y `unit_price::float8 AS unit_price` a la query de `account_statement_items`, falló porque la tabla `account_statement_items` NO tiene esas columnas en la DB. El `cargo build` pasó porque sqlx usa macros que pueden no validar en compile time si la DB no está disponible.
**Fix**: crear migración `030_add_quantity_unit_price_to_statement_items.sql` que agrega `quantity NUMERIC(10,2)` y `unit_price NUMERIC(10,2)` a `account_statement_items`. Ambas nullable (NULL para extras conceptuales). El endpoint de Extras de Enfermería (`addItem`) no necesita cambios — enviará NULL automáticamente.

## Bug 3 — Paquetes Aplicados no muestra Precio Unit. 
Misma causa que Bug 2: `unit_price` no llega desde el backend porque la columna no existe en DB. Se resuelve con la migración del Bug 2.

## Bug 4 — PDF no muestra columnas de cantidad para items de paquete/bodega
El PDF se genera en el backend. Revisar si el endpoint `GET /api/admissions/{id}/pdf` incluye quantity/unit_price en su output. Si el backend ya los obtiene de `admission_extras` (para bodega) y necesita también de `account_statement_items` (para paquetes), agregar los campos al PDF template una vez que la migración del Bug 2 esté lista. Si el PDF ya los obtiene de otra fuente, verificar qué falta.

## Bug 5 — Eliminar botón "Imprimir"
En `AdmissionDetailPage.vue`, eliminar el botón "Imprimir" del header. Ya existe "Descargar PDF". Solo queda ese.
