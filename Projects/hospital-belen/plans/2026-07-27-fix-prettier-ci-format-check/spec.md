# Spec — Fix Prettier format-check CI failure (web module)

**Project:** hospital-belen · **effort:** low · **stage-1**
**Scope:** `hospital-belen-web/kairosaid`. CI job `lint` → step `Format check (Prettier)`
= `npm run format:check` (`prettier --check .`). Pre-existing on `main`.

## Failing files (5)

`prettier --check .` reports:

| File | Issue (all pure Prettier canonical reflow — no logic) |
|------|------|
| `src/modules/profile/pages/ProfilePage.vue` | Template attribute wrapping differs from Prettier's canonical form (collapses/rewraps multi-line element tags). |
| `src/modules/profile/services/profileService.ts` | Long return-type union `{…}\|{…}` not broken onto its own lines. |
| `src/services/api.service.ts` | Long boolean `return` expr not wrapped in parens across lines; general spacing. |
| `src/stores/platform.store.ts` | Minor whitespace / trailing-newline diff (near-identical). |
| `src/utils/roleColor.ts` | Long array literal (`PALETTE`) and long function param list not broken across lines. |

None involve behavior. All are standard Prettier line-wrapping / spacing.

## Fix approach

Single command — let Prettier rewrite the files to its canonical form:

```bash
cd hospital-belen-web/kairosaid
./node_modules/.bin/prettier --write \
  src/modules/profile/pages/ProfilePage.vue \
  src/modules/profile/services/profileService.ts \
  src/services/api.service.ts \
  src/stores/platform.store.ts \
  src/utils/roleColor.ts
```

(There is no `format` write script — only `format:check`; call the binary directly,
or add `"format": "prettier --write ."` if the team wants one. Don't run `--write .`
across the whole tree in this pass — keep the diff to the 5 files.)

## Caveat / do-not-fight

- `ProfilePage.vue` reflow will look like it contradicts the `vue/max-attributes-per-line`
  ESLint **warnings** — that's fine. Those are warnings (don't fail CI); Prettier owns
  formatting here. Do **not** hand-edit to satisfy both. Prettier wins.
- Do not touch logic, imports, or template semantics — only run the formatter.

## Verification

```bash
npm run format:check   # → "All matched files use Prettier code style!"
npm run lint           # still 0 errors (warnings unchanged)
```

## Out of scope

- No ESLint rule changes.
- No reformat of files Prettier already accepts.
