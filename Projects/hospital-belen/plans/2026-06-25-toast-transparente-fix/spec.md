# Spec — Fix toast/sonner transparente en hospital-belen-web

## Resumen ejecutivo del diagnóstico

**Causa raíz: el proyecto `hospital-belen-web/kairosaid`, NO la librería `@inksightdev/ui`.**

La librería expone el componente `Sonner` correctamente y su CSS interno define backgrounds opacos válidos para light theme:

- Toast neutro: `[data-sonner-toaster][data-theme=light]` → `--normal-bg:#fff` (blanco puro, opaco).
- Toast con `rich-colors`: `[data-rich-colors=true][data-sonner-toast][data-type=success|info|warning|error]` → backgrounds con lightness 96-97% (ej: `--success-bg:hsl(143,85%,96%)`, `--info-bg:hsl(208,100%,97%)`, `--warning-bg:hsl(49,100%,97%)`, `--error-bg:hsl(359,100%,97%)`).

El problema visual: el proyecto define `--background: 210 20% 94%` (page bg ~94% lightness, casi blanco azulado). El `<RouterLayoutView>` pasa `rich-colors` al Sonner. Los backgrounds rich-colors del toast (96-97% L) se confunden visualmente con el body bg (94% L). Resultado: parecen "transparentes" / casi invisibles aunque el CSS no usa transparencia.

Secundario (no es la causa, pero conviene anotar): `package.json` del proyecto declara `vue-sonner@^2.0.9` pero `@inksightdev/ui@0.3.18` depende internamente de `vue-sonner@^1.3.0` (vendoreado en su `dist/index.js`). El `vue-sonner` raíz nunca se importa desde `src/` — es dependencia muerta. No es la causa del bug, pero el Coder NO debe tocarla (fuera de scope).

**Conclusión:** el fix vive en `hospital-belen-web/kairosaid` (capa de uso/tema), no en `@inksightdev/ui`. La librería se comporta como está documentada. El problema es que el sistema de color del proyecto tiene un page background tan claro que choca con los presets rich-colors.

---

## Archivos a modificar

### 1. `/Users/hp/Documents/tony/projects/hospital-belen/hospital-belen-web/kairosaid/src/assets/main.css`

Agregar un bloque al final del archivo (fuera de `@layer base` y `@layer components` para que la especificidad no la coma Tailwind ni `:where(...)` del CSS inyectado por vue-sonner).

Objetivo del bloque: sobrescribir las variables `--*-bg`, `--*-border`, `--*-text` y `--normal-bg` que Sonner usa internamente, para que las superficies de toast sean **opacas y separadas del page bg**. La sobrescritura se hace en el selector raíz del toaster `[data-sonner-toaster][data-theme=light]` (mayor especificidad que `:where(...)` y empata la especificidad del bloque interno del paquete, pero al venir DESPUÉS en orden de cascada gana).

Requisitos exactos del bloque:

- Selector base: `[data-sonner-toaster][data-theme=light]`
- Redefinir los siguientes custom properties con valores opacos y con contraste claro contra `hsl(210 20% 94%)`:
  - `--normal-bg`: blanco puro `#ffffff` (mantener), pero **agregar también** `--normal-border: hsl(175 20% 85%)` (un border teal-tinted más visible que el `--gray4` del paquete) para definir el borde del toast neutral.
  - `--success-bg`: bajar la lightness de 96% a **88%** → `hsl(143 65% 88%)`. Border: `hsl(143 60% 70%)`. Text: mantener `hsl(140 100% 22%)` (más oscuro que el preset para AA contrast).
  - `--info-bg`: `hsl(208 90% 90%)`. Border: `hsl(208 80% 72%)`. Text: `hsl(210 92% 35%)`.
  - `--warning-bg`: `hsl(45 95% 88%)`. Border: `hsl(40 85% 70%)`. Text: `hsl(28 92% 35%)`.
  - `--error-bg`: `hsl(0 90% 92%)`. Border: `hsl(0 85% 78%)`. Text: `hsl(0 80% 38%)`.
- Para el modo dark, agregar también un selector hermano `[data-sonner-toaster][data-theme=dark]` con los mismos custom properties, pero usando los valores que YA tiene el paquete (no modificarlos — el problema es solo light). Esto es opcional pero recomendado por simetría; si quieres minimalismo, omítelo.
- Adicionalmente, reforzar el shadow para diferenciar el toast del fondo claro: dentro del bloque agregar la regla:
  ```
  [data-sonner-toaster][data-theme=light] [data-sonner-toast][data-styled=true] {
    box-shadow: 0 8px 24px -4px rgba(15, 118, 110, 0.18), 0 4px 8px -2px rgba(15, 23, 42, 0.08);
  }
  ```
  El RGB `15, 118, 110` es el teal primary del proyecto (`hsl(175 84% 32%)` convertido), dando una sombra sutilmente brand-tinted.

**Patrón a seguir:** los bloques existentes en `main.css` (especialmente el override de `:-webkit-autofill` líneas 132-145) ya usan el patrón de "override global de componente de tercero vía CSS plano fuera de `@layer`". Copiar esa misma estructura y ubicación (al final del archivo).

### 2. `/Users/hp/Documents/tony/projects/hospital-belen/hospital-belen-web/kairosaid/src/layouts/RouterLayoutView.vue`

**No modificar el archivo.** El uso actual del componente es correcto:
```vue
<Sonner position="top-right" rich-colors :close-button="true" :duration="4000" />
```
La prop `rich-colors` se mantiene — el fix funciona porque ahora los rich-colors-bg que sobrescribimos sí contrastan.

---

## Archivos NO modificar (scope guard)

- `node_modules/@inksightdev/ui/**` (terceros, gestionado vía npm)
- `package.json` — NO remover `vue-sonner` del proyecto aunque sea dead dep (fuera de scope; puede ser otro ticket)
- Cualquier otro componente que use `toast.success|error|warning|info`
- `tailwind.config.js`
- `RouterLayoutView.vue` — no agregar más props ni `:toast-options`

---

## Diseño visual

**Subject:** sistema de notificaciones para un hospital (Hospital Belén). Audiencia: personal clínico y administrativo trabajando en una UI densa de citas, pacientes y permisos. El toast tiene un solo trabajo: confirmar o alertar sobre la última acción sin robar foco del flujo principal.

**Palette del toast (light theme, derivado del brand teal `hsl(175 84% 32%)`):**

- Neutral surface: `#FFFFFF` con border `hsl(175 20% 85%)` (teal-tinted hairline) y sombra teal sutil. La sombra es la firma — distingue el toast del page bg sin necesidad de elevar la saturación del fill.
- Success: verde mint sólido (`hsl(143 65% 88%)`), texto verde oscuro AA-legible.
- Info: azul cielo sólido (`hsl(208 90% 90%)`), texto azul medio.
- Warning: amarillo cálido sólido (`hsl(45 95% 88%)`), texto ámbar oscuro.
- Error: rosa coral sólido (`hsl(0 90% 92%)`), texto rojo oscuro.

**Decisión de contraste:** se bajó la lightness de los presets de Sonner ~8 puntos (96% → 88%) porque sobre un page bg de 94% L, cualquier superficie >92% L se lee como "casi transparente". 88% L da separación clara sin perder la sensación "light/airy" que el proyecto ya tiene.

**Firma:** la sombra teal-tinted (`rgba(15, 118, 110, 0.18)`) en lugar de la sombra negra genérica del default Sonner. Es el único elemento brand-identitario del toast, y conecta visualmente con el color primary del producto sin gritar.

**Typography y layout:** intactos. El componente Sonner del paquete ya define type-scale e iconos; no se tocan. La restricción es deliberada — toda la opinión visual del fix vive en color + shadow.

---

## Edge cases que el Coder debe cubrir

1. **Modo dark:** el proyecto tiene `.dark` definido en `main.css`. El Sonner del paquete ya define un theme dark propio (`[data-sonner-toaster][data-theme=dark]`) con backgrounds oscuros que SÍ contrastan con `--background: 0 0% 7%`. No tocar dark. El override se aplica únicamente al selector `[data-sonner-toaster][data-theme=light]`.
2. **Especificidad:** el bloque DEBE ir DESPUÉS del `@layer components` y fuera de cualquier `@layer`. Vue-sonner inyecta su CSS vía JS al `document.head` en runtime — eso significa que el CSS de Sonner se appendea DESPUÉS de los stylesheets importados. Por eso necesitamos especificidad mayor o igual al selector original. `[data-sonner-toaster][data-theme=light]` (2 selectores de atributo, specificity 0,2,0) empata al selector del paquete; al venir después en orden de aparición, sí gana — PERO solo si nuestro CSS se inyecta DESPUÉS del JS de Sonner. Como Sonner inyecta al mount del componente y nuestro CSS se carga al boot vía `main.ts`, **el nuestro carga ANTES**. Solución: usar `!important` en cada propiedad redeclarada de los custom properties. Las custom properties con `!important` sí se respetan correctamente en CSS cascade. Sin `!important`, el orden source pierde y los valores del paquete ganan.
3. **`rich-colors` activado:** el componente se usa con `rich-colors`. Validar que las clases `[data-rich-colors=true][data-sonner-toast][data-type=success|info|warning|error]` reciban los nuevos backgrounds. Como esas reglas en el paquete usan `background:var(--success-bg)` etc., al sobrescribir la custom property heredada desde `[data-sonner-toaster]`, los hijos heredan automáticamente.
4. **Close button:** el toast tiene `:close-button="true"`. El close button del paquete usa `background:var(--gray1)` / `color:var(--gray12)`. Esos custom properties siguen viniendo del paquete y son neutros — no requieren override. NO tocar.
5. **Mobile (<600px):** el paquete ya tiene media query mobile. No tocar.
6. **prefers-reduced-motion:** ya soportado por el paquete. No tocar.

---

## Validación (qué debe revisar el Tester)

1. Disparar manualmente cada variante desde la consola del browser en una página autenticada:
   ```js
   // pegar en devtools console
   const { toast } = await import('@inksightdev/ui')
   toast.success('Cita creada correctamente')
   toast.info('Recordatorios programados')
   toast.warning('Conflicto detectado en horario')
   toast.error('Sin conexión al servidor')
   toast('Notificación neutral sin tipo')
   ```
2. Verificar visualmente que cada toast tiene un background **claramente distinguible** del page bg `hsl(210 20% 94%)` — ya no se ve "transparente".
3. Verificar que el texto del toast es legible (contraste AA mínimo).
4. Verificar que el shadow teal-tinted es visible bajo el toast.
5. Tomar screenshot a `/Users/hp/Documents/tony/projects/hospital-belen/.pipeline/screenshots/toast-after.png`.
6. Ejecutar `npm run build` para confirmar que no rompió tipos/build.

---

## Patrón a seguir (referencia)

Mirar el bloque `input:-webkit-autofill` en `src/assets/main.css` líneas 132-145 — usa exactamente el mismo patrón: CSS plano fuera de `@layer` con `!important` para vencer estilos de Chromium/terceros. El fix del toast sigue esa misma plantilla mental.

---

## Notas para el Reviewer

- Cambio mínimo: 1 archivo modificado (`main.css`), ~30-40 líneas agregadas al final.
- No se instalan dependencias.
- No se modifica el componente ni sus props.
- El uso de `!important` en custom properties está justificado por el orden de inyección runtime de vue-sonner (ver edge case #2).
- Decisión consciente: NO se eliminó la dep huérfana `vue-sonner@^2.0.9` del package.json (fuera de scope del ticket).
