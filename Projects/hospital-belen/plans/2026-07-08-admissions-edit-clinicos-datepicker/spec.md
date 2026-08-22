# Spec — Admissions: edit Datos Clínicos + Talonario DatePicker

Repo: `hospital-belen` · frontend `hospital-belen-web/kairosaid` · Vue 3 + `@inksightdev/ui` + Tailwind. effort=medium.

## ESTADO ACTUAL (leer antes de trabajar)

Ambas features **ya están implementadas en el working tree** (sin commit). `git status`:
- `M src/modules/admissions/pages/AdmissionDetailPage.vue` — feature 1 (pencil + modal edit).
- `M src/modules/admissions/pages/AdmissionListPage.vue` — DatePicker en filtros (referencia).
- `M src/modules/admissions/services/admissionService.ts` — `update()` para guardar.
- `M src/modules/receipts/components/ReceiptForm.vue` — feature 2 (swap `<input type=date>` → `<DatePicker>`).

El trabajo REAL restante = **el bug de background transparente del DatePicker** (feature 2, punto 2b).

---

## Feature 1 — Botón edit Datos Clínicos → YA HECHO, solo verificar

Implementado en `AdmissionDetailPage.vue`:
- Lápiz `<Pencil>` (lucide) en header del Card "Datos Clínicos" (líneas 74-82), `variant="ghost" size="icon"`, clase `text-muted-foreground` (gris, ya responsivo light/dark vía token). Oculto si `status === 'CLOSED'`.
- Modal `<Dialog>` `max-w-2xl` (líneas 314-373) con los campos de step 1 del create wizard.
- Campos = los de `AdmissionCreatePage.vue` "Información de Admisión": `admissionType`, `procedureName`, `room`, `doctorName`, `assistantName`, `surgeonName`, `anesthesiologistName`, `preliminaryDiagnosis`.
- Guarda vía `admissionService.update(id, {...})` + `adm.fetchAdmission()` (líneas 909-932).

**Acción coder:** verificar que compila (`npm run build`) y que el modal abre/edita/guarda. No reimplementar.

OPEN QUESTION: step 1 del create NO incluye paciente (Card aparte) ni fecha de admisión. El modal replica solo "Información de Admisión". ¿"datos step 1" = solo esos campos clínicos, sin paciente/fecha? Implementación actual asume que sí.

---

## Feature 2 — Talonario DatePicker

### 2a. Swap del selector → YA HECHO, solo verificar
`ReceiptForm.vue` líneas 24-29: `<input type="date">` reemplazado por `<DatePicker v-model="dateObj" locale="es-GT" placeholder="Fecha">`. Puente `dateObj` (Date ↔ string `YYYY-MM-DD`) maneja timezone con partes locales. Verificar submit del form.

### 2b. Background transparente — DIAGNÓSTICO: es el PACKAGE

**Causa raíz (confirmada):** el `DatePicker` de `@inksightdev/ui` renderiza su `PopoverContent` con:
```
class: "w-[var(--radix-popover-trigger-width)] p-0"
```
→ **sin `bg-popover` ni `border`**. Los demás popovers del package (Select, Combobox, Dialog) sí traen `bg-popover border`. Por eso el calendario se ve transparente.

- `--popover` SÍ está definido en `src/assets/main.css` (light `0 0% 100%`, dark `0 0% 10%`) — no es problema de tokens nuestro.
- La utility `.bg-popover` existe (tailwind app + `@inksightdev/ui/styles`) — no falta CSS.
- El defecto es markup del package: el DatePicker omite el fondo en su content.

**Veredicto para el humano:** bug del package `inksightdev-components`, no nuestro. Afecta a TODOS los DatePicker (admissions filters + receipts).

### Fix (app-side, sin tocar node_modules)
Override CSS en `src/assets/main.css` — hay precedente exacto (bloque Sonner usa `!important` para ganar cascada de CSS runtime). Añadir regla que target el popover del DatePicker:

```css
/* DatePicker (@inksightdev/ui) renderiza su PopoverContent sin bg-popover/border.
   Bug del package; parche aquí hasta fix upstream. */
[data-reka-popper-content-wrapper] .w-\[var\(--radix-popover-trigger-width\)\] {
  background-color: hsl(var(--popover));
  border: 1px solid hsl(var(--border));
  border-radius: var(--radius, 0.5rem);
}
```

Edge case / verificación obligatoria: confirmar el selector exacto en devtools con el DatePicker abierto — reka-ui puede usar otro wrapper (`data-reka-popper-content-wrapper` vs `data-radix-*`) y la clase va escapada. El coder inspecciona el DOM real antes de fijar el selector. Ceiling: si el package publica fix, borrar el override.

**Además:** anotar issue upstream contra `inksightdev-components` (DatePicker PopoverContent sin `bg-popover border`). No bloqueante.

---

## Archivos
- Modificar: `hospital-belen-web/kairosaid/src/assets/main.css` (añadir override, 2b).
- Verificar (sin cambios salvo bugs): `AdmissionDetailPage.vue`, `ReceiptForm.vue`, `AdmissionListPage.vue`, `admissionService.ts`.

## Patrones a seguir
- Override CSS: mismo estilo que el bloque Sonner en `main.css` (comentario + selector con especificidad suficiente).
- Sin dependencias nuevas. Sin editar `node_modules`.

## Diseño visual
Sin diseño nuevo — restaurar fondo del popover a los tokens existentes (`--popover` / `--border`). Debe verse igual que Select/Combobox del mismo package en light y dark.

## Test
- `npm run build` limpio.
- Manual: abrir DatePicker en ReceiptForm y AdmissionList → fondo sólido opaco light+dark, borde visible.
- Manual: pencil en Datos Clínicos → modal abre, edita, guarda, refresca.

SKILL_RECOMENDADA: ninguna (fix de estilo puntual).
