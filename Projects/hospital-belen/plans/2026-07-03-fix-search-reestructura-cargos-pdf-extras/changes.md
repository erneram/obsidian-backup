# changes.md — AdmissionDetailPage: 6 fixes
**project:** hospital-belen  
**dispatch:** stage-2  
**date:** 2026-07-04

---

## Archivos modificados

### `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue`

- **Issue 1a**: `<Command :shouldFilter="false">` en Popover de bodega — evita que Command filtre por UUID en lugar de nombre
- **Issue 1b**: Selector de paquetes reemplazado por Popover+Command con `pkgComboOpen`, `pkgQuery`, `filteredPackages` computed; `openApplyPackage()` resetea query y open
- **Issue 2**: Card "Cargos de Inventario" eliminada; card "Paquetes Aplicados" unificada (siempre visible) con dos sub-secciones: tabla packageItems + sub-sección "Cargos de Inventario" (extras)
- **Issue 3**: "Extras de Enfermería" → "Extras" (CardTitle + DialogTitle)
- **Issue 5**: Ambos `PopoverContent class="w-full p-0"` → `w-[--radix-popper-anchor-width] p-0` en el dialog de inventario (bodega + ítem)

### `hospital-belen-api/src/web/handlers/admission.rs`

- **Issue 6a**: Formato PDF de paquetes: `(x5 @ Q12.50)` → `· 5 × Q12.50`

### `hospital-belen-api/src/infrastructure/pdf/mod.rs`

- **Issue 6b**: Concept arrays ampliados para cubrir todos los enum values del frontend:
  - HOSPITALIZACIÓN: agrega `HOSPITALIZACION`
  - OTROS GASTOS: agrega `OTROS`, `MEDICAMENTOS`, `MATERIAL_QUIRURGICO`
  - LABORATORIOS: agrega `LABORATORIO`, `RAYOS_X`, `ULTRASONIDO`
  - HONORARIOS MÉDICOS: agrega `HONORARIOS_MEDICOS`, `QUIROFANO`, `ANESTESIA`

---

## Qué debe revisar el Tester

1. **Bodega search**: escribir en el input de bodega filtra por nombre (no por UUID)
2. **Package combobox**: escribir filtra la lista; selección carga los items del paquete
3. **Layout unificado**: card "Paquetes Aplicados" muestra sub-sección de paquetes y sub-sección "Cargos de Inventario"; no hay card separada
4. **Rename**: "Extras" en CardTitle y DialogTitle
5. **PopoverContent width**: dropdowns de bodega e ítem tienen el mismo ancho que el botón trigger
6. **PDF formato**: items de paquete muestran `· N × Q12.50` en el PDF
7. **PDF conceptos**: crear admisión con items LABORATORIO, HONORARIOS_MEDICOS, QUIROFANO, etc. y verificar que aparecen en el PDF bajo sus secciones
