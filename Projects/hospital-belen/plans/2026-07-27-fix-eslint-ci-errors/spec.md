# Spec — Fix ESLint CI failures (web module)

**Project:** hospital-belen · **effort:** medium · **stage-1**
**Scope:** `hospital-belen-web/kairosaid` only. CI job: `.github/workflows/ci-frontend.yml` → `lint` (`npm run lint` = `eslint . --ext .vue,.js,...`, no `--max-warnings`).

## What actually fails CI

ESLint exits non-zero only on **errors**, not warnings (no `--max-warnings` flag).
Scanning **source only** (`src/`), there are exactly:

- **2 errors** — `vue/html-self-closing` ← **this is the CI blocker**
- **491 warnings** — do not fail the build (informational)

> Note on the "816 errors" seen locally: that count comes from ESLint also linting
> `dist/assets/*.js` (minified build output). `dist/` is **not** git-tracked and the
> CI lint job does **not** build, so those never occur in CI. They are a local-only
> artifact of a stale `dist/`. Still worth fixing via config (Priority 2).

## Priority 1 — Fix the 2 blocking errors (makes CI green)

Rule `vue/html-self-closing` config: `html.normal: 'never'` → native elements must
**not** self-close. Two `<span … />` violate it:

| File | Line | Fix |
|------|------|-----|
| `src/modules/platform/components/TenantAccordion.vue` | 149 | `<span … />` → `<span …></span>` |
| `src/modules/platform/pages/PlatformRolesPage.vue` | 51 | `<span … />` → `<span …></span>` |

`PlatformRolesPage.vue:51` is:
```html
<span v-if="!r.is_superadmin" class="w-2.5 h-2.5 rounded-full shrink-0" :style="{ backgroundColor: roleColor(r) }" />
```
→ close explicitly: `… :style="{ backgroundColor: roleColor(r) }"></span>`

Simplest: `npx eslint src/modules/platform/components/TenantAccordion.vue src/modules/platform/pages/PlatformRolesPage.vue --fix` (auto-fixes this rule), then confirm `npm run format:check` (prettier) still passes on the two files — prettier agrees with the expanded form, but verify.

## Priority 2 — Ignore build output in eslint config (hygiene, prevents 816-error trap)

**File:** `eslint.config.js`. Flat config's default only ignores `node_modules/` and
`.git/`; `dist/` is linted. Add a top-of-array ignores block so a local build before
lint (or any future CI step order change) doesn't flood 800+ false positives from
minified bundles:

```js
export default [
  { ignores: ['dist/**', 'coverage/**', 'node_modules/**'] },
  js.configs.recommended,
  ...
]
```

Zero behavior change for CI today; removes the entire minified-bundle noise class
(`no-unused-vars`, `no-undef`, `no-cond-assign`, `no-fallthrough`, `no-useless-escape`,
`no-control-regex` — all from `dist/`).

## Priority 3 — Warnings (OUT of scope for the CI fix; documented by impact)

491 warnings, none fail CI. Do **not** bulk-fix in this pass. Ranked:

| Rule | Count | Nature | Recommendation |
|------|-------|--------|----------------|
| `vue/max-attributes-per-line` | 251 | Pure formatting, auto-fixable | Separate cosmetic PR (`--fix`); big diff, risk of prettier overlap. Or turn the rule `off` and let Prettier own line-wrapping. |
| `vue/no-restricted-html-elements` | 227 | Deliberate design-system rule: native `<button>/<input>/<table>/…` should use `@inksightdev/ui` | Real migration debt, NOT a lint-config problem. Own effort, module by module. Keep as `warn`. |
| `@typescript-eslint/no-explicit-any` | 12 | Real typing gaps | Fix case-by-case later. |
| `vue/no-template-shadow` | 1 | Real shadowing | Trivial, opportunistic. |

**Do not** flip the `lint` script to `--max-warnings 0` — it would convert all 491
into build failures. Leave warnings as warnings.

## Existing patterns / references

- Rule severities and the `@inksightdev/ui` enforcement list live in `eslint.config.js`
  (`UI_ELEMENT_RULES`, lines 14–67). Calendar components are already exempted
  (lines 142–147) — follow that exemption pattern if any future exemption is needed.

## Out of scope

- No dist changes (build artifact).
- No design-system migration (`no-restricted-html-elements`).
- No `--max-warnings` change.

## Verification

- `npm run lint` from `kairosaid/` (with `dist/` absent, as in CI) → 0 errors.
- `npm run format:check` → passes.
- After Priority 2, a stray `dist/` no longer inflates the count.
