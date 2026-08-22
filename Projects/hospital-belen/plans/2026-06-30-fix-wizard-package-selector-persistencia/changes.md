# Changes — estado-cuenta wizard package selector UX fix

**Implementado:** 2026-06-30  
**Modo:** stage-2

---

## Archivos modificados

### `estado-cuenta/components/StepItems.vue`

- **Trigger label**: muestra `ec.pendingPackages.value[0]?.packageName` cuando hay un paquete seleccionado; "Buscar paquete..." si no hay ninguno
- **`onSelectPackage`**: reemplaza `splice(existing,1,entry) / push(entry)` por `splice(0, Infinity, entry)` → solo un paquete pendiente a la vez; seleccionar otro reemplaza el anterior

### `estado-cuenta/components/StepReview.vue`

- **`onMounted`**: removido el loop de `applyPackage` y el clear de `pendingPackages` — ya no aplica automáticamente al montar
- **`applying` ref + `applyPending()`**: nueva función que itera `pendingPackages`, llama `ec.applyPackage` por cada uno, limpia el array solo si tiene éxito (try/finally — en error el paquete persiste para reintentar)
- **Template**: nueva `<Card v-if="ec.pendingPackages.value.length">` antes del card de items — muestra package name + tabla de ítems (qty × unitPrice) + botón "Aplicar paquete" explícito

---

## Comportamiento post-fix

1. Seleccionar paquete B estando A seleccionado: B reemplaza A (no se acumulan)
2. Items → Review → Back: `pendingPackages` intacto, vista expandida con `[- N +]` visible
3. En Review: card "Paquete por aplicar" muestra breakdown + botón explícito
4. Al aplicar: `pendingPackages` se limpia, items PACKAGE aparecen en la lista del estado de cuenta
5. Si `applyPackage` falla: `pendingPackages` no se limpia, usuario puede reintentar

---

## Qué revisar (Tester)

- Step 2: seleccionar paquete → aparece inmediatamente expandido con `[- N +]`
- Cambiar paquete: el anterior desaparece y aparece el nuevo
- Trigger del combobox muestra nombre del paquete seleccionado
- Step 3: si hay paquete pendiente, card "Paquete por aplicar" visible con detalle
- Step 3 → Back → Step 3: paquete pendiente sigue ahí
- "Aplicar paquete" en review: cargos PACKAGE aparecen en tabla de items, card desaparece
- Error en apply: card persiste, mensaje de error visible
