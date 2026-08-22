# Changes — Fix Prettier format-check CI failure (web module)

## Approach

Ran `prettier --write` on exactly the 5 files reported by `prettier --check .`. Pure formatting — zero logic changes.

## Files reformatted

- `src/modules/profile/pages/ProfilePage.vue` — template attribute wrapping
- `src/modules/profile/services/profileService.ts` — long return-type union line break
- `src/services/api.service.ts` — boolean return expr, spacing
- `src/stores/platform.store.ts` — minor whitespace / trailing newline
- `src/utils/roleColor.ts` — PALETTE array and param list line breaks

## Verification results

- `npm run format:check` → "All matched files use Prettier code style!" ✓
- `npm run lint` → 0 errors, 491 warnings ✓ (unchanged from previous fix)

## Fix round — Prettier/ESLint conflict on self-closing spans

**Root cause**: Prettier collapses empty `<span></span>` to `<span />` in Vue templates;
ESLint `vue/html-self-closing` (html.normal: never) forbids self-closing on normal elements.

### Files touched
- `src/modules/platform/components/TenantAccordion.vue` (line 149)
- `src/modules/platform/pages/PlatformRolesPage.vue` (line 51)

**Fix**: Added `<!-- prettier-ignore -->` on the line preceding each conflicting span so
Prettier skips reformatting them. ESLint still sees the valid `<span></span>` form.

### Post-fix verification
- `npm run format:check` → "All matched files use Prettier code style!" ✓
- `eslint src/` → 0 errors, 491 warnings ✓ (814-error count locally is `dist/` artifact, not CI)

## Repos tocados

- **web** (`hospital-belen-web`): branch `fenix/fix-prettier-format`, 2 commits total, pushed.

## Tester should verify

1. `npm run format:check` from `kairosaid/` → passes.
2. `npm run lint` (without dist/) → 0 errors, 491 warnings.
3. No behavior changes — template structure and business logic identical to pre-format state.
