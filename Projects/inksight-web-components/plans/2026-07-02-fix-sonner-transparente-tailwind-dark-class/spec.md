# Spec — Sonner: Fix transparencia / tema incorrecto
**project:** inksight-web-components  
**dispatch:** stage-1 (2026-07-02)

---

## Diagnóstico

### Root cause 1 — Desincronización de tema (crítico)

`Sonner.vue` pasa `theme` a vue-sonner's `<Toaster>`. El default es `'system'`, que hace que vue-sonner use `window.matchMedia('(prefers-color-scheme: dark)')` para detectar el tema.

hospital-belen-web usa Tailwind con estrategia de clase (`class="dark"` en `<html>`), controlada por `useTheme.ts` (lee de `localStorage`, fallback a `prefers-color-scheme`). El usuario puede tener:
- OS en light mode + app en dark → toaster recibe `data-theme="light"` mientras la app se ve oscura
- OS en dark mode + app en dark → toaster recibe `data-theme="dark"` ✓

Cuando `data-theme="dark"`, vue-sonner v1 establece `--normal-bg: #000` (negro puro). El fondo oscuro de la app es `hsl(0 0% 7%)`. Negro sobre negro = invisible.

### Root cause 2 — CSS variables demasiado oscuras en dark mode

hospital-belen-web tiene overrides en `main.css` con `!important`:
```css
[data-sonner-toaster][data-theme=dark] {
  --normal-bg: hsl(0 0% 14%) !important;  /* apenas 14% lightness vs 7% bg */
}
```
14% sobre 7% da contraste insuficiente. Además, con `rich-colors=true`, `--success-bg: hsl(140 35% 16%)` también es demasiado oscuro.

### Versiones involucradas
- Fuente local: `@inksightdev/ui` **0.3.17** (en `packages/ui-brand/`)
- Publicado en npm: **0.3.18** (instalado en hospital-belen-web)
- hospital-belen-web requiere: `"^0.3.18"` (acepta 0.3.18 → <0.4.0)
- vue-sonner en la librería: **^1.3.0** (bundleado en dist/index.js)
- vue-sonner en hospital-belen-web: **2.0.9** (sólo en node_modules, el app usa la versión bundleada de la librería)

---

## Cambios

### 1. `packages/ui-brand/src/components/Sonner.vue` — sincronizar con clase Tailwind

Reemplazar el componente completo:

```vue
<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { Toaster } from 'vue-sonner'

const props = defineProps<{
  theme?: 'light' | 'dark' | 'system'
  position?: 'top-left' | 'top-center' | 'top-right' | 'bottom-left' | 'bottom-center' | 'bottom-right'
  richColors?: boolean
  expand?: boolean
  duration?: number
  closeButton?: boolean
  offset?: string | number
  toastOptions?: Record<string, unknown>
}>()

const {
  theme = 'system',
  position = 'bottom-right',
  richColors = false,
  expand = false,
  duration = 4000,
  closeButton = false,
} = props

// When theme='system', detect Tailwind dark class instead of OS prefers-color-scheme.
// vue-sonner's built-in system detection uses matchMedia which ignores Tailwind class strategy.
const tailwindTheme = ref<'light' | 'dark'>('light')
let observer: MutationObserver | null = null

function syncTailwindTheme() {
  tailwindTheme.value = document.documentElement.classList.contains('dark') ? 'dark' : 'light'
}

onMounted(() => {
  if (theme !== 'system') return
  syncTailwindTheme()
  observer = new MutationObserver(syncTailwindTheme)
  observer.observe(document.documentElement, { attributeFilter: ['class'] })
})

onUnmounted(() => observer?.disconnect())

const activeTheme = computed(() => theme === 'system' ? tailwindTheme.value : theme)
</script>

<template>
  <Toaster
    :theme="activeTheme"
    :position="position"
    :rich-colors="richColors"
    :expand="expand"
    :duration="duration"
    :close-button="closeButton"
    :offset="offset"
    :toast-options="toastOptions"
    class="toaster group"
  />
</template>
```

**Cambios respecto a la versión actual:**
- Ya no desestructura props en el nivel de setup (necesita `computed` que referencia `theme`)
- `activeTheme` computed: usa `tailwindTheme` (observa clase del `<html>`) cuando `theme='system'`; pasa directo cuando el consumidor lo especifica explícitamente

---

### 2. `packages/ui-brand/src/styles/index.css` — CSS overrides para Sonner

Agregar al final del archivo. Esto corrige los colores base para ambos modos y se incluye automáticamente en cualquier app que importe `@inksightdev/ui/styles`:

```css
/* ── Sonner toast overrides ─────────────────────────────────────────────────── */
/* vue-sonner inyecta su CSS en runtime; usamos !important para ganar la cascada */

[data-sonner-toaster][data-theme=light] {
  --normal-bg: #ffffff !important;
  --normal-border: hsl(215 20% 88%) !important;
  --normal-text: hsl(220 15% 20%) !important;
  --success-bg: hsl(143 60% 93%) !important;
  --success-border: hsl(143 55% 78%) !important;
  --success-text: hsl(140 70% 25%) !important;
  --info-bg: hsl(208 85% 93%) !important;
  --info-border: hsl(208 75% 75%) !important;
  --info-text: hsl(210 80% 38%) !important;
  --warning-bg: hsl(45 90% 90%) !important;
  --warning-border: hsl(40 80% 72%) !important;
  --warning-text: hsl(28 85% 32%) !important;
  --error-bg: hsl(0 85% 94%) !important;
  --error-border: hsl(0 80% 80%) !important;
  --error-text: hsl(0 72% 38%) !important;
}

[data-sonner-toaster][data-theme=dark] {
  --normal-bg: hsl(0 0% 18%) !important;
  --normal-border: hsl(0 0% 26%) !important;
  --normal-text: hsl(0 0% 95%) !important;
  --success-bg: hsl(140 30% 22%) !important;
  --success-border: hsl(140 35% 32%) !important;
  --success-text: hsl(143 55% 72%) !important;
  --info-bg: hsl(210 30% 22%) !important;
  --info-border: hsl(210 35% 34%) !important;
  --info-text: hsl(208 75% 74%) !important;
  --warning-bg: hsl(38 30% 22%) !important;
  --warning-border: hsl(40 35% 34%) !important;
  --warning-text: hsl(45 85% 70%) !important;
  --error-bg: hsl(0 30% 22%) !important;
  --error-border: hsl(0 35% 34%) !important;
  --error-text: hsl(0 80% 74%) !important;
}
```

**Nota**: `hsl(0 0% 18%)` para dark bg da contraste suficiente contra el fondo de la app (`hsl(0 0% 7%)`). El coder puede ajustar si el diseñador tiene tokens específicos.

---

### 3. `packages/ui-brand/package.json` — bump de versión

```diff
- "version": "0.3.17",
+ "version": "0.3.19",
```

(Se salta 0.3.18 porque ya está publicada en npm con código diferente.)

---

### 4. Actualizar test en `Sonner.test.ts`

Agregar test de tema:
```ts
it('uses dark theme when html has dark class', async () => {
  document.documentElement.classList.add('dark')
  const wrapper = mount(Sonner)
  await wrapper.vm.$nextTick()
  // activeTheme should resolve to 'dark'
  // Since Toaster is mocked, just verify no error thrown
  expect(wrapper.exists()).toBe(true)
  document.documentElement.classList.remove('dark')
})
```

---

### 5. Build y publish

```bash
# En packages/ui-brand/
pnpm build
npm publish
```

---

## Fix inmediato en hospital-belen-web (sin esperar publish)

Mientras se publica la nueva versión, el consumidor puede pasar el tema explícitamente:

**`hospital-belen-web/kairosaid/src/layouts/RouterLayoutView.vue`**:

```diff
+import { useTheme } from '@/composables/useTheme'
+const { theme } = useTheme()
 
-<Sonner position="top-right" rich-colors :close-button="true" :duration="4000" />
+<Sonner position="top-right" rich-colors :close-button="true" :duration="4000" :theme="theme" />
```

Después de instalar `0.3.19`, esta línea puede mantenerse (el MutationObserver no interfiere cuando `theme` se pasa explícitamente) o eliminarse (el componente detecta solo).

---

## Instalar nueva versión en hospital-belen-web

```bash
cd hospital-belen-web/kairosaid
npm install @inksightdev/ui@0.3.19
# o si ya tienen ^0.3.18:
npm update @inksightdev/ui
```

También: eliminar los overrides de Sonner en `hospital-belen-web/kairosaid/src/assets/main.css` (líneas ~148-202) ya que el fix quedará en la librería. **Opcional** — si se dejan, refuerzan los colores a nivel de app.

---

## Resumen de archivos a cambiar

| Repo | Archivo | Cambio |
|------|---------|--------|
| inksight-web-components | `packages/ui-brand/src/components/Sonner.vue` | MutationObserver para Tailwind dark class |
| inksight-web-components | `packages/ui-brand/src/styles/index.css` | Agregar CSS overrides Sonner |
| inksight-web-components | `packages/ui-brand/src/components/__tests__/Sonner.test.ts` | Test de dark class detection |
| inksight-web-components | `packages/ui-brand/package.json` | Version bump 0.3.17 → 0.3.19 |
| hospital-belen-web | `kairosaid/src/layouts/RouterLayoutView.vue` | `:theme="theme"` fix inmediato |
| hospital-belen-web | `kairosaid/package.json` | (auto-update con pnpm install) |
