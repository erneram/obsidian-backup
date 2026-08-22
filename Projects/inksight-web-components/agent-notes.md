
## 2026-07-02 — Fix Sonner/toaster transparente

- Decisión: MutationObserver en `document.documentElement` para sincronizar clase Tailwind `.dark` — no depender del OS `prefers-color-scheme`
- Decisión: CSS overrides con `!important` en `styles/index.css` — 30 variables (15 light + 15 dark)
- Version bump: 0.3.17 → 0.3.19 (0.3.18 ya publicada con código diferente)
- Fix inmediato en hospital-belen-web: `:theme="theme"` en RouterLayoutView (funciona sin esperar publish)
- Deuda: `const { theme } = props` rompe reactividad Vue 3 — fix en próximo ciclo si se necesita prop dinámica
- Blockers: npm publish requiere login manual
- Siguiente paso: `npm login` + `npm publish` + `npm install @inksightdev/ui@0.3.19` en hospital-belen-web
