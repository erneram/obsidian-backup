# Test Results — Fix toast/sonner transparente

## Veredicto: PASS

---

## Tests corridos

### 1. CSS static validation (`test-toast-css.mjs`)

**30 / 30 assertions passed**

Suites:
- **block structure** (3): override fuera de `@layer`, shadow selector fuera de `@layer`, override no anidado en `@layer`.
- **custom properties con !important** (14): `--normal-bg`, `--normal-border`, `--success-*`, `--info-*`, `--warning-*`, `--error-*` todos declarados con `!important`.
- **color values match spec** (6): `#ffffff`, `hsl(175 20% 85%)`, `hsl(143 65% 88%)`, `hsl(208 90% 90%)`, `hsl(45 95% 88%)`, `hsl(0 90% 92%)` presentes y correctos.
- **brand-tinted shadow** (2): `rgba(15, 118, 110, 0.18)` y `rgba(15, 23, 42, 0.08)` presentes.
- **dark mode guard** (2): no hay bloque `[data-theme=dark]` en el override; `.dark {}` en `@layer base` intacto.
- **edge cases** (3): selector con 2 atributos (specificity 0,2,0); `--normal-bg` no usa rgba con alpha; `box-shadow` presente.

### 2. `vite build` (producción)

**Resultado: SUCCESS** — 102 chunks generados, `dist/` completo.

El bloque override quedó incluido en `dist/assets/index-BOaszpzO.css`:
```
[data-sonner-toaster][data-theme=light]{--normal-bg: #ffffff !important; ...}
```

### 3. `npm run build` (`vue-tsc -b && vite build`)

**vue-tsc falla** con 9 errores de TypeScript en archivos NO relacionados con este cambio:
- `AddAppointmentModal.vue` — TS2352 en cast de tipo
- `PatientFormStep1.vue` — TS2345 en prop `class: string[]`
- `MainLayout.vue` — TS6133 (unused import), TS2322, TS2352
- `SearchBarLayout.vue` — TS2352
- `UserListPage.vue` — TS2345
- `CalendarPage.vue` — TS6133 (unused import)
- `GrowthPointForm.vue` — TS2339

**Estos errores son pre-existentes y están fuera del scope del cambio.** El archivo modificado (`src/assets/main.css`) no tiene componente TypeScript. Vite build (el paso que incluye y transforma CSS) pasa sin errores.

---

## Nota para el Reviewer

Los errores de `vue-tsc` existían antes del fix y no son regresiones introducidas por este cambio. El CSS del override compiló correctamente y está presente en el bundle de producción. El fix de CSS es correcto según el spec.

Pendiente (manual, requiere browser):
- Visual check de cada variante de toast contra page bg `hsl(210 20% 94%)`.
- Screenshot a `.pipeline/screenshots/toast-after.png`.
