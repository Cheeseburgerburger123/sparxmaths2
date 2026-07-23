# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This repo builds **SparxSolver V2** (internal codename **Azalea**), a Chrome/Edge/Opera MV3 browser
extension that patches the Sparx Maths web app (`*.sparx-learning.com`) client-side: it hooks into
the site's own React tree to bypass "bookwork check" answer gates, capture/store answers locally,
and add a themeable in-app settings menu. Based on/forked from
[acquitelol/sparxmaths](https://github.com/acquitelol/sparxmaths).

This is a **client-side patching/injection tool that runs entirely in the end user's browser
against a site the user is already using** — there is no scraping infrastructure, no bulk
automation, and no interaction with systems other than the browser tab the extension is installed
in. Treat it like any other browser-extension codebase.

## Build system & commands

Package manager is **pnpm** (see CI in `.github/workflows/build.yml`; `pnpm-lock*` is gitignored).

```bash
pnpm i          # install deps
pnpm build      # rollup -c --configPlugin esbuild -> dist/
pnpm watch      # pnpm build --watch
```

`rollup.config.ts` defines two IIFE bundles, both minified with esbuild and then obfuscated with
`javascript-obfuscator`:

- `src/index.ts` → `dist/index.js` — the MV3 **content script** (runs at `document_start`,
  injected per `manifest.json`'s `content_scripts`). It just injects `core.css`/`cute.css` and loads
  `bundle.js` into the page.
- `src/entry/index.ts` → `dist/bundle.js` — the actual page-context bundle (uses `nodeResolve` +
  `commonjs` for React/KaTeX deps). A `copy-extension` rollup plugin also copies everything from
  `extension/` (manifest, CSS, `assets/`, `headers.json`, `themes.json`, `index.html`) into `dist/`
  after build, so `dist/` ends up as a complete loadable unpacked extension.
- Path aliases (`@core`, `@handlers`, `@modules`, `@patches`, `@components`, `@utilities`, `@entry`,
  `@extension`, `@logger`, `@stylesheet`) are defined in `tsconfig.json` (`baseUrl: src`) and
  resolved at build time via `rollup-plugin-typescript-paths`. Ambient/global-only aliases
  (`@azalea/types`, `@azalea/components`, `@azalea/themes`, `@azalea/settings`, `@azalea/bookwork`,
  `@azalea/buttons`, `@azalea/utilities`) are **type-only** `declare module` stubs in
  `src/types/types.d.ts` — they exist purely for TypeScript's benefit and don't need a matching
  rollup alias.
- `src/types/global.d.ts` declares the injected `globalThis.azalea` object (the return value of
  `core/index.ts`'s `generateAzalea()`) and the host page's own `window.__sparxweb` shape.

**CI** (`.github/workflows/build.yml`, runs on every push, `macos-latest`): installs pnpm, runs
`pnpm build`, zips `dist/` as `Azalea-<version>.zip` (version read from
`extension/manifest.json`), and publishes a GitHub Release tagged with that version via
`softprops/action-gh-release`. **Bumping `extension/manifest.json`'s `version` is what drives a new
release** — do this deliberately, not incidentally.

There is no test suite configured in this repo.

## Loading the extension locally

1. `pnpm i && pnpm build`
2. Chrome/Edge/Opera → `chrome://extensions` (or equivalent) → enable Developer Mode → "Load
   unpacked" → select `dist/`.
3. Visit a `*.sparx-learning.com` page.

## Linting

ESLint config is `.eslintrc.js` (not flat config): `plugin:react/recommended` +
`@typescript-eslint`, with `react/react-in-jsx-scope` off (React is globally exfiltrated, not
imported per-file — see below). Enforced style: **single quotes** (JS and JSX) and **required
semicolons**. There's no lint script in `package.json`; run `npx eslint src` directly if needed.
`.markdownlint.json` lints Markdown (`README.md`) — default rules with `MD013` (line length)
disabled.

## Architecture

### Injection flow

`src/index.ts` (content script, `document_start`) → injects stylesheets and loads
`src/entry/index.ts` (`bundle.js`) into the **page's own JS context** (not an isolated content-script
world) → `entry/index.ts` runs `initializeRoutes`, `initializeWindow`, `initializePrefs`,
`initializeLogo` in parallel via `allSettledValidated`, each wrapped in `validate()` (from
`entry/validate.ts`) so one failing initializer doesn't take down the rest.

### Core (`src/core/`)

- `core/index.ts` — `generateAzalea()` assembles the single global `azalea` object
  (`modules`, `components`, `handlers`, `utilities`, `patches`, `patcher`, `hooks`, `navigation`,
  `version`) exposed as `window.azalea`/`globalThis.azalea`.
- `core/patcher.ts` — thin re-export of the [`spitroast`](https://www.npmjs.com/package/spitroast)
  React-patching library (`before`/`after`/`instead` style function patching), used throughout
  `patches/` to hook into the host site's React render functions.
- `core/modules/exfiltrate.ts` — a clever prototype-pollution trick (credited to
  [uwu/shelter](https://github.com/uwu/shelter)) that grabs internal module references (React,
  ReactDOM, a `useMediaQuery` hook) straight out of the host page's own bundled webpack modules by
  temporarily hooking `Object.prototype` property assignment. `core/modules/data.ts` lists which
  prop name + type-guard filter to use for each exfiltrated module (`MediaQuery`/`React`/`ReactDOM`).
  **This is inherently fragile against upstream Sparx bundle changes** — if patches stop finding
  React/module references, check here first.
- `core/handlers/` — small stateful utility classes: `StorageHandler` (`storage.ts`, a thin
  namespaced `localStorage` JSON key/value wrapper with `set`/`get`/`delete`/`toggle`/`list`/`clear`),
  `state.ts` (the shared `storages` singleton map: `colors`, `preferences`, `bookwork`, `updater`,
  each a separate `localStorage` namespace like `AzaleaPreferences`), and `Theming` (`theming.ts`,
  applies theme color CSS custom properties — `--<colorType>-<key>` — to `documentElement`, reading
  the active theme from `preferences.get('themeIndex')` and merging built-in themes from
  `extension/themes.json` plus a synthesized `'Custom'` theme backed by the `colors` storage).
- `core/utilities/` — DOM/React tree traversal helpers used by patches: `findReact`/`findInReactTree`/
  `findInTree` (walk React fiber/prop trees to locate specific components), `lazyDefine` (poll until
  a DOM node/condition exists), `navigate`, `chunkArray`, `isEmpty`, `common.ts`.
- `core/components/` — shared UI primitives (`buttons.tsx`, `dividers.tsx`, `section.tsx`,
  `row.tsx`) plus a small `katex/` subsystem (`renderLatex`, `replaceLatex`, `preprocess`,
  `htmlEscape`) for rendering LaTeX math (`TextWithMaths`) inside injected UI, since bookwork
  answers frequently contain math notation.
- `core/stylesheet.ts` — `createStyleSheet`/`commonStyles` helper for building reusable inline-style
  objects (a lightweight CSS-in-JS convention used across `patches/`, e.g.
  `commonStyles.merge(x => [x.flex, x.row, {...}])`).

### Patches (`src/core/patches/`)

Each patch is a self-contained async function returning an optional cleanup (e.g. a
`MutationObserver.disconnect`), all run in parallel and independently error-isolated by
`Promise.allSettled` in `patches/index.ts`:

- `bookworkBypass.tsx` — the core feature. Watches the page's `#root` for the "Work Answer Check"
  (WAC) React component, exfiltrates its previously-cached answers from the `bookwork` storage
  keyed by bookwork code, renders a `BookworkSection` showing the last 3 stored attempts (including
  image answers hosted on `assets.sparxhomework.uk` and LaTeX text answers), and — if
  `preferences.get('autoBookwork')` is enabled — auto-selects the matching multiple-choice option by
  calling the host component's own `onSelect` handler.
- `captureAnswers.ts` — presumably the counterpart that writes newly-submitted answers into the
  `bookwork` storage (check this file directly when modifying answer-capture behavior).
- `menuButtons.tsx` / `menu/` — injects the in-app settings menu UI: `menu/settings/` (About,
  Toggles, name/logo customization inputs, theme color pickers under `themes/`) and
  `menu/bookwork/` (Answer/Bookwork/Section/Toggle/Listing components for browsing stored answers).

### Extension shell (`extension/`)

Static MV3 assets copied verbatim into `dist/` at build time: `manifest.json` (permissions:
`declarativeNetRequest` + `storage`; host permissions scoped to `sparx-learning.com` and its `auth`
subdomain), `headers.json` (a `declarativeNetRequest` rule set that strips the host site's CSP so
the injected bundle/styles can run), `themes.json` (built-in theme definitions consumed by
`Theming`), `core.css`/`cute.css` (base + default theme styles), `index.html` (unused/legacy?
verify before relying on it), and `assets/` (logo, divider image).

## Key conventions

- **Path aliases over relative imports.** Use the `@core`/`@handlers`/`@modules`/`@patches`/
  `@components`/`@utilities`/`@logger`/`@stylesheet`/`@extension` aliases (see `tsconfig.json`)
  rather than deep relative paths like `../../../core/utilities`.
- **Never import React/ReactDOM directly.** They're exfiltrated at runtime from the host page's own
  bundle (`core/modules/exfiltrate.ts` + `data.ts`) and exposed via `core/modules` (`common.React`,
  etc.) — importing a separate React instance would break interop with the host site's component
  tree and hooks (`patcher.after('render', ...)` patches rely on identity).
  Consequently `react/react-in-jsx-scope` is disabled in ESLint.
- **Patch, don't replace.** New host-site integrations should use `patcher` (`spitroast`) to hook
  existing React render functions (`before`/`after`/`instead`) rather than trying to
  remove/replace/re-render host components outright — the extension has to coexist with Sparx's own
  React tree indefinitely and survive its re-renders.
- **Every patch is independently fault-tolerant.** New entries in `patches/index.ts` (and
  initializers in `entry/index.ts`) must be wrapped so a thrown error in one patch/initializer
  can't break the others — follow the existing `Promise.allSettled` + `validate()` pattern.
- **Persistence is `localStorage`, namespaced per concern.** Add new persisted state via a new (or
  existing) `StorageHandler` instance registered in `core/handlers/state.ts`'s `storages` map —
  don't call `localStorage` directly from patch code.
- **Style with the `createStyleSheet`/`commonStyles` helper**, not inline object literals scattered
  ad hoc, and prefer existing CSS custom properties (`var(--colours-selected)`,
  `var(--palette-white)`, theme-driven `--<type>-<key>` variables) over hardcoded colors so the
  theming system (`Theming.setTheme`) keeps working.
- **Code style:** single quotes, required semicolons, 4-space indentation (see existing files) —
  matches `.eslintrc.js`.
- **Version bumps drive releases.** `extension/manifest.json`'s `version` field is read by CI to tag
  and publish a GitHub Release on every push to any branch — only bump it when you intend to cut a
  release.
