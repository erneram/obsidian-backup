# Changes — Fix ESLint CI failures (web module)

## Priority 1 — 2 blocking errors fixed

### `kairosaid/src/modules/platform/components/TenantAccordion.vue` (line 149)
- `<span … />` → `<span …></span>` — fixes `vue/html-self-closing` error.

### `kairosaid/src/modules/platform/pages/PlatformRolesPage.vue` (line 51)
- `<span … />` → `<span …></span>` — fixes `vue/html-self-closing` error.

## Priority 2 — eslint.config.js ignores block

### `kairosaid/eslint.config.js`
- Added `{ ignores: ['dist/**', 'coverage/**', 'node_modules/**'] }` as first entry in the default export array.
- Prevents minified `dist/` output from inflating lint error count with 800+ false positives.
- Zero behavior change for CI (dist is not present during CI lint job).

## Verification result

`npm run lint` (from `kairosaid/`) → **0 errors, 491 warnings** in 74 files. ✓

## Repos tocados

- **web** (`hospital-belen-web`): branch `fenix/fix-eslint-errors`, 1 commit, pushed.

## Not changed

- No warning rules touched.
- No `--max-warnings` change.
- No design-system migration.
- `dist/` not touched.

## Tester should verify

1. `npm run lint` from `kairosaid/` → 0 errors.
2. `npm run format:check` → passes (prettier agrees with expanded `<span></span>` form).
3. With a stray `dist/` present: `npm run lint` still shows 0 errors (ignores block active).
