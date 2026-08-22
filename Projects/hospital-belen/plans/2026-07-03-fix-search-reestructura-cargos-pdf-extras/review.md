VERDICT: SHIP

## Razonamiento

Los 6 issues implementados conforme al spec. `:shouldFilter="false"` en los 3 Commands correctos, `filteredPackages` computed correcto, card "Cargos de Inventario" eliminada y unificada como sub-sección, rename "Extras" completo, concept arrays cubren todos los valores del enum en español, PDF formato `· N × Q12.50`. Sin imports ni refs huérfanos.

## Hallazgos

🟢 BAJO: `:shouldFilter="false"` presente en líneas 361 (bodega), 387 (ítem), 461 (paquetes) — los 3 Commands que lo necesitan ✅
🟢 BAJO: `filteredPackages` computed (l.959-962) filtra `availablePackages` por `pkgQuery` case-insensitive; `pkgComboOpen` se resetea a `false` al abrir el dialog — flujo correcto ✅
🟢 BAJO: Card unificada (l.127-208): sub-sección packageItems + sub-sección "Cargos de Inventario" sin card separada. `v-else` muestra "Sin paquetes aplicados." cuando no hay ítems ✅
🟢 BAJO: CardTitle → "Extras" (l.211), DialogTitle → "Agregar Extra" (l.252) ✅
🟢 BAJO: `PopoverContent class="w-[--radix-popper-anchor-width] p-0"` en ambos popovers del dialog de inventario (l.360, 386) ✅
🟢 BAJO: PDF format `· {} × Q{:.2}` (admission.rs l.324) ✅
🟢 BAJO: Concept arrays — `HOSPITALIZACION`, `LABORATORIO`/`RAYOS_X`/`ULTRASONIDO`, `HONORARIOS_MEDICOS`/`QUIROFANO`/`ANESTESIA`, `OTROS`/`MEDICAMENTOS`/`MATERIAL_QUIRURGICO` — todos cubiertos (pdf/mod.rs l.670-692) ✅
🟢 BAJO: Sin imports huérfanos — `Trash2`, `Receipt`, `FileDown`, `Minus`, `ArrowLeft`, `ChevronsUpDown` todos usados en template ✅
🟡 BAJO: Comentario interno l.838 dice "Extras de Enfermería" — cosmético, no usuario-visible.
