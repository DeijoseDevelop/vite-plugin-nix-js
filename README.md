# @deijose/vite-plugin-nix-js

Vite plugin for Nix.js with Hot Module Replacement (HMR) and session preservation.

## Requirements

- Vite `^8.0.0`
- `@deijose/nix-js` `^2.5.1`

## What it does

This plugin gives Nix.js applications a development experience similar to React Fast Refresh, Vue HMR, or Svelte HMR:

- **Hot reload** of components without full page refresh.
- **Preserves state** of stores, routers, forms, and signals.
- **Preserves scroll position** and focused element.
- **Zero manual wrapping** required from the developer.

## Installation

```bash
npm install -D @deijose/vite-plugin-nix-js
# or
pnpm add -D @deijose/vite-plugin-nix-js
# or
yarn add -D @deijose/vite-plugin-nix-js
```

## Usage

```ts
// vite.config.ts
import { defineConfig } from "vite";
import nix from "@deijose/vite-plugin-nix-js";

export default defineConfig({
  plugins: [nix()],
});
```

That is all the developer needs to write.

## How it works

### Step 1: Detect Nix.js imports

The plugin scans each source file for imports from `@deijose/nix-js` and tracks which of these functions are used locally:

- `createStore`
- `createRouter`
- `mount`

### Step 2: Transform code at build-time

When the plugin detects these functions, it transforms the code automatically. The developer writes:

```ts
const cart = createStore({ items: [] }, { name: "cart" });
const router = createRouter(routes);
mount(App(), "#app", { router });
```

The transformed code (never seen by the developer) becomes:

```ts
import { __nixGetOrCreateStore, __nixGetOrCreateRouter, __nixMount, __nixHmrAccept } from "@deijose/vite-plugin-nix-js/runtime";

const cart = __nixGetOrCreateStore("src/main.ts:cart", () => createStore({ items: [] }, { name: "cart" }));
const router = __nixGetOrCreateRouter("src/main.ts:router", () => createRouter(routes));
__nixMount("src/main.ts", () => App(), "#app", { router });

if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    __nixHmrAccept(newModule, "src/main.ts");
  });
}
```

### Step 3: Runtime keeps state alive

The runtime module exports small helper functions that live in a global singleton attached to `window.__nixHmrRuntime`.

- `__nixGetOrCreateStore(id, factory)`: returns the existing store if it was already created, otherwise creates it.
- `__nixGetOrCreateRouter(id, factory)`: same for routers.
- `__nixMount(id, factory, container, options)`: mounts the app the first time, and re-mounts it when HMR fires.
- `__nixHmrAccept(newModule, id)`: saves scroll/focus snapshot, unmounts the old component, re-mounts with the new factory, and restores the snapshot.

### Step 4: Vite HMR triggers the update

When a source file changes, Vite calls `import.meta.hot.accept`. The plugin-injected handler calls the runtime, which re-renders the affected component while preserving global state.

## Why the core is not touched

- The template cache uses a `WeakMap` keyed by `TemplateStringsArray`. Each HMR reload provides a new `TemplateStringsArray`, so the cache naturally refreshes without invalidation.
- Signals and stores are plain objects that the runtime can read/write through `$snapshot()` and `$patch()`.
- The router exposes `current`, `params`, `query`, and the history stack.
- No changes to `nix-js-microframework` are required.

## Supported cases

- **Multiple mount points** in the same file. Each `mount()` gets a unique id and all are re-mounted on HMR.
- **Mount assigned to a variable** (`const handle = mount(...)`) as well as bare `mount(...)` statements.
- **Module-scoped stores and routers**, including named exports (`export const cart = createStore(...)`).
- **Async components** (`mount(await loadApp(), "#app")`). The generated factory is async and the runtime awaits the resolved component before mounting.

Stores and routers declared **inside functions** are intentionally NOT registered globally, because they are meant to produce a fresh instance on each call.

## Limitations

- Component-level state preservation (as opposed to global store/router state) is not yet implemented.
- Only top-level `mount` calls are tracked; mounts nested inside component render functions are left untouched.

## Development

```bash
cd vite-plugin-nix
npm install
npm run typecheck
npm run build
```

## Roadmap

- [ ] Preserve form state with `createForm`.
- [ ] Preserve router history stack.
- [ ] Preserve component-level state for `NixComponent` classes.
- [ ] Browser extension devtools integration.
- [ ] Multi-mount support.

## License

MIT
