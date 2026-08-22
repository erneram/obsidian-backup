# Spec — AdmissionDetailPage: 6 fixes
**project:** hospital-belen  
**dispatch:** stage-1 (2026-07-04)

---

## Archivos a modificar

| Repo | Archivo | Issues |
|------|---------|--------|
| hospital-belen-web | `kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue` | 1, 2, 3, 5 |
| hospital-belen-api | `src/web/handlers/admission.rs` | 6a |
| hospital-belen-api | `src/infrastructure/pdf/mod.rs` | 6b |

---

## Issue 1 — Search/filter no funciona

### Root cause 1a: Storage combobox filtra por UUID, no por nombre

El `<Command>` del storage selector no tiene `:shouldFilter="false"`. El componente
Command intenta filtrar los `<CommandItem>` usando su prop `:value` (un UUID).
Al escribir "bodega quirúrgica", Command trata de hacer match contra el UUID
`abc-123-...` → no encuentra nada → oculta todos los items.

`invFilteredStorages` ya hace el filtro client-side correctamente; solo falta
desactivar el filtro interno de Command.

**Fix — `AdmissionDetailPage.vue` línea ~363:**
```diff
-<Command>
+<Command :shouldFilter="false">
```

### Root cause 1b: Package selector usa `<Select>` sin búsqueda

`openApplyPackage()` carga todos los paquetes activos en `availablePackages` y los
muestra en un `<Select>`. Sin filtro, con 10+ paquetes el usuario no puede buscar.

**Fix — reemplazar `<Select>` de paquetes con Combobox:**

Agregar `pkgQuery = ref('')` y computed:
```ts
const filteredPackages = computed(() =>
  pkgQuery.value.trim()
    ? availablePackages.value.filter(p =>
        p.name.toLowerCase().includes(pkgQuery.value.toLowerCase())
      )
    : availablePackages.value
)
```

Reemplazar en template (líneas 453–462):
```html
<!-- antes: <Select v-model="selectedPkgId" @update:model-value="onPackageSelected"> -->
<Popover v-model:open="pkgComboOpen">
  <PopoverTrigger as-child>
    <Button variant="outline" role="combobox" class="w-full justify-between">
      {{ availablePackages.find(p => p.id === selectedPkgId)?.name ?? '-- Seleccionar paquete --' }}
      <ChevronsUpDown class="h-4 w-4 opacity-50" />
    </Button>
  </PopoverTrigger>
  <PopoverContent class="w-[--radix-popper-anchor-width] p-0">
    <Command :shouldFilter="false">
      <CommandInput placeholder="Buscar paquete..." v-model="pkgQuery" />
      <CommandEmpty>Sin paquetes</CommandEmpty>
      <CommandGroup class="max-h-48 overflow-y-auto">
        <CommandItem
          v-for="p in filteredPackages"
          :key="p.id"
          :value="p.id"
          @select="() => { selectedPkgId = p.id; pkgComboOpen = false; onPackageSelected(p.id) }"
        >{{ p.name }} (Q{{ Number(p.baseCost).toFixed(2) }})</CommandItem>
      </CommandGroup>
    </Command>
  </PopoverContent>
</Popover>
```

Agregar `pkgComboOpen = ref(false)` en el script.  
Eliminar las variables `pkgQuery` y `filteredPackages` si existían antes (o crearlas si no).

---

## Issue 2 — Restructurar "Cargos de Inventario"

**Objetivo:** eliminar la card separada "Cargos de Inventario"; moverla como
sub-sección dentro de "Paquetes Aplicados". El botón "Agregar cargo" se mueve
al header de la nueva card unificada.

### Eliminar la card "Cargos de Inventario" (líneas 127–168 actuales)

Borrar el bloque completo:
```html
<!-- Cargos de Inventario (extras only) -->
<Card>
  <CardHeader class="flex flex-row items-center justify-between">
    <CardTitle class="text-base">Cargos de Inventario</CardTitle>
    <Button ...>Agregar cargo</Button>
  </CardHeader>
  <CardContent>...</CardContent>
</Card>
```

### Reemplazar la card "Paquetes Aplicados" (líneas 170–206)

Cambia de `v-if="packageItems.length"` a siempre visible (la sección de extras
también necesita mostrarse). Nueva card unificada:

```html
<Card>
  <CardHeader class="flex flex-row items-center justify-between">
    <CardTitle class="text-base">Paquetes Aplicados</CardTitle>
    <Button
      v-if="detail.statement.status !== 'CLOSED'"
      size="sm"
      @click="openAddInventoryCharge"
    >
      <Plus :size="14" class="mr-1" /> Agregar cargo
    </Button>
  </CardHeader>
  <CardContent class="space-y-4">
    <!-- Sub-sección: ítems de paquetes -->
    <div v-if="packageItems.length">
      <Table class="min-w-full divide-y divide-border text-sm">
        <!-- ... misma tabla de packageItems de antes ... -->
      </Table>
    </div>
    <p v-else class="text-sm text-muted-foreground">Sin paquetes aplicados.</p>

    <!-- Sub-sección: cargos de inventario (extras) -->
    <div>
      <p class="text-xs font-medium text-muted-foreground uppercase tracking-wide mb-2">
        Cargos de Inventario
      </p>
      <div v-if="extrasLoading" class="py-2 text-center text-muted-foreground text-sm">Cargando...</div>
      <Table v-else class="min-w-full divide-y divide-border text-sm">
        <!-- ... misma tabla de extras de antes (Descripción | Cant. | Precio Unit. | Total | 🗑) ... -->
      </Table>
    </div>
  </CardContent>
</Card>
```

---

## Issue 3 — Rename "Extras de Enfermería" → "Extras"

Dos ocurrencias en `AdmissionDetailPage.vue`:

```diff
-<CardTitle class="text-base">Extras de Enfermería</CardTitle>
+<CardTitle class="text-base">Extras</CardTitle>
```

```diff
-<DialogTitle>Agregar Extra de Enfermería</DialogTitle>
+<DialogTitle>Agregar Extra</DialogTitle>
```

---

## Issue 4 — Steppers [- N +]: verificado correcto

Código actual en inventory modal: `invQty = Math.max(1, invQty - 1)` / `invQty++`  
Código actual en package modal: `Math.max(1, (pkgItemQty[id] ?? qty) - 1)` / `(pkgItemQty[id] ?? qty) + 1`

Ambos son +1/-1. **No se requiere cambio de código.**

---

## Issue 5 — Modal overflow al seleccionar bodega

### Root cause

`PopoverContent class="w-full p-0"` — en un Portal, `w-full` hereda del
`<body>` (100vw), no del trigger. Cuando el Popover se despliega dentro de
un Dialog, el contenido puede solapar bordes o mostrarse con ancho incorrecto.

### Fix — ambos PopoverContent dentro del dialog de inventario

```diff
-<PopoverContent class="w-full p-0">
+<PopoverContent class="w-[--radix-popper-anchor-width] p-0">
```

Aplica tanto al Popover de bodega (línea ~360) como al Popover de ítem (línea ~386).

`--radix-popper-anchor-width` es la CSS var que Radix establece automáticamente
igual al ancho del trigger — garantiza que el dropdown tenga exactamente el ancho
del botón que lo abre.

---

## Issue 6 — PDF

### 6a — Cambiar formato `@ Q` → `× Q`

**`hospital-belen-api/src/web/handlers/admission.rs` línea 324:**

```diff
-(Some(q), Some(p)) => format!("{} (x{} @ Q{:.2})", i.description, q, p),
+(Some(q), Some(p)) => format!("{} · {} × Q{:.2}", i.description, q, p),
```

Resultado: `"Aspirina · 5 × Q12.50"` en lugar de `"Aspirina (x5 @ Q12.50)"`.

---

### 6b — Extras (Hospitalización, Laboratorio, etc.) no aparecen en PDF

**Root cause:** Las concepts stored en DB por el frontend son
`HOSPITALIZACION`, `LABORATORIO`, `HONORARIOS_MEDICOS`, etc. El PDF en
`pdf/mod.rs` busca `HOSPITAL`, `LABORATORY`, `FEES` — strings distintos.
Ningún item conceptual coincide → sección vacía → `ec_section` retorna sin imprimir.

**Fix — `hospital-belen-api/src/infrastructure/pdf/mod.rs` línea ~670:**

```diff
// HOSPITALIZACIÓN / EXTRAS
-let mut hosp = collect(&["HOSPITAL"]);
+let mut hosp = collect(&["HOSPITAL", "HOSPITALIZACION"]);
 for e in &data.extras { ... }
 ec_section(&mut doc, "HOSPITALIZACIÓN / EXTRAS", &hosp)?;

// OTROS GASTOS
-ec_section(&mut doc, "OTROS GASTOS", &collect(&["OTHER", "ROOM", "ADVANCE"]))?;
+ec_section(&mut doc, "OTROS GASTOS", &collect(&[
+    "OTHER", "ROOM", "ADVANCE",
+    "OTROS", "MEDICAMENTOS", "MATERIAL_QUIRURGICO",
+]))?;

// LABORATORIOS
-ec_section(&mut doc, "LABORATORIOS", &collect(&["LABORATORY"]))?;
+ec_section(&mut doc, "LABORATORIOS", &collect(&[
+    "LABORATORY", "LABORATORIO", "RAYOS_X", "ULTRASONIDO",
+]))?;

// HONORARIOS MÉDICOS
-ec_section(&mut doc, "HONORARIOS MÉDICOS", &collect(&["FEES"]))?;
+ec_section(&mut doc, "HONORARIOS MÉDICOS", &collect(&[
+    "FEES", "HONORARIOS_MEDICOS", "QUIROFANO", "ANESTESIA",
+]))?;
```

**Nota:** `OTROS` (inventoryItems creados vía `createExtra`) ya aparece en el PDF
vía `data.extras` (ExtraPdfLine). No agregar `OTROS` a paquete / hospitalización
para evitar duplicados.

---

## Edge cases

- Package combobox: si `availablePackages` está vacío, mostrar `CommandEmpty`.
- `--radix-popper-anchor-width` requiere Radix Vue ≥ 1.x — confirmar que la versión instalada lo soporta; si no, usar `style="width: max-content; min-width: 100%"` como fallback.
- Extras re-clasificados en PDF: ítems con concepto `MEDICAMENTOS`/`MATERIAL_QUIRURGICO`/`RAYOS_X`/`ULTRASONIDO` aparecerán por primera vez en el PDF — revisar que los totales de sección sean correctos en prueba manual.
