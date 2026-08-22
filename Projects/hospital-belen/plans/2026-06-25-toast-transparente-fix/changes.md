# Changes — Fix toast/sonner transparente

## Archivos modificados

### `hospital-belen-web/kairosaid/src/assets/main.css`
Bloque agregado al final del archivo, fuera de `@layer base` y `@layer components`.

Contiene:
- `[data-sonner-toaster][data-theme=light]` — sobrescribe custom properties `--normal-bg`, `--normal-border`, `--success-bg/border/text`, `--info-bg/border/text`, `--warning-bg/border/text`, `--error-bg/border/text` con `!important`. Cada color bajado ~8 puntos de lightness respecto al preset del paquete (96-97% L → 88-92% L) para contrastar con el page bg `hsl(210 20% 94%)`.
- `[data-sonner-toaster][data-theme=light] [data-sonner-toast][data-styled=true]` — sombra teal-tinted brand `rgba(15, 118, 110, 0.18)` para separar el toast del fondo visualmente.
- El `!important` es necesario porque vue-sonner inyecta su CSS al runtime (después del boot), y nuestro CSS carga antes.

## Qué debe revisar el Tester

1. En una página autenticada, pegar en devtools console:
   ```js
   const { toast } = await import('@inksightdev/ui')
   toast.success('Cita creada correctamente')
   toast.info('Recordatorios programados')
   toast.warning('Conflicto detectado en horario')
   toast.error('Sin conexión al servidor')
   toast('Notificación neutral sin tipo')
   ```
2. Cada toast debe tener background **claramente distinguible** del page bg — ya no "transparente".
3. Texto legible con contraste AA mínimo en cada variante.
4. Shadow teal-tinted visible bajo el toast.
5. Screenshot a `.pipeline/screenshots/toast-after.png`.
6. `npm run build` sin errores de tipos/build.
7. Verificar que dark mode no cambió (el override solo aplica a `[data-theme=light]`).
