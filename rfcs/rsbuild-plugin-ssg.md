# RFC: rsbuild-plugin-ssg

- Feature Name: `rsbuild_plugin_ssg`
- Start Date: 2026-11-06
- RFC PR: TBD
- Issue: TBD

## Summary

`@computerwwwizards/rsbuild-plugin-ssg` pre-renders a chosen list of routes of a React app at build time. Every other route keeps rendering in the browser from an empty shell.

## Motivation

The target applications render fully on the client. The browser downloads and runs JavaScript before it paints any content. That delay hurts user-centric metrics such as First Contentful Paint (FCP) and Largest Contentful Paint (LCP).

Pre-rendering the critical routes removes that delay, within these constraints:

- **No extra cost.** The budget doesn't cover a rendering server, so nothing renders at request time.
- **No longer builds.** Rendering every route at build time would grow build times, so the plugin renders only the chosen routes.
- **React only.** The applications use React.
- **Each project chooses its routes.** A project lists the routes to pre-render. Every other route renders in the browser.
- **Soft navigation stays.** After the first load, the client-side router handles navigation.
- **Any router.** The plugin doesn't depend on a specific client-side router.

## Guide-level explanation

Import `pluginSsg` and configure its two required options:

- `prerender` lists the routes to pre-render. It needs at least one route.
- `server` points to the module that exports the factory.

```ts
// rsbuild.config.ts
import { defineConfig } from '@rsbuild/core';
import { pluginReact } from '@rsbuild/plugin-react';
import { pluginSsg } from '@computerwwwizards/rsbuild-plugin-ssg';

export default defineConfig({
  plugins: [
    pluginReact(),
    pluginSsg({
      prerender: ['/', '/contact'],
      server: './ssg/server.tsx',
    }),
  ],
});
```

The server module exports the factory as `factoryServerFunction`. The plugin calls the factory once per build, so it's the place for one-time setup, such as loading translations. The factory resolves to a render function. The plugin calls the render function once per route, with the route's URL, and pre-renders the React tree it resolves to. You decide how each route renders on the server, so the plugin isn't tied to a router.

```tsx
// ssg/server.tsx
import { Router } from 'wouter';
import App from '../src/App';

export const factoryServerFunction = async () => {
  return async (url: string) => (
    <Router ssrPath={url}>
      <App />
    </Router>
  );
};
```

The client entry is a regular client-side rendered app, read from `source.entry` in the Rsbuild config. It doesn't change:

```tsx
// src/index.tsx
import { createRoot } from 'react-dom/client';
import { Router } from 'wouter';
import App from './App';

createRoot(document.getElementById('root')!).render(
  <Router>
    <App />
  </Router>,
);
```

The server and the client must build the same React tree. If they don't, React reports a hydration mismatch in the browser when the page loads.

After a successful build, the output directory contains these files:

```text
index.html          pre-rendered home page
contact.html        pre-rendered contact page
spa.html            shell with an empty container
static/...          JS and CSS chunks
ssg-manifest.json   which file serves each route
```

```json
{
  "version": 1,
  "prerendered": { "/": "index.html", "/contact": "contact.html" },
  "fallback": "spa.html"
}
```

### Loading data before rendering

Two places can wait for asynchronous work. The factory runs once and receives every route in `routes`, so it can load data for all of them in one request. The render function runs once per route, so it can load data that depends on that URL.

Both functions receive an `AbortSignal`. Pass it to `fetch` or to your data library. When a route runs out of time, the plugin aborts the signal, and the work that listens to it stops.

```tsx
export const factoryServerFunction = async ({ routes, signal }) => {
  // Once per build. `routes` lists every route the plugin renders.
  const cache = await preload(routes, { signal });

  return async (url, { signal }) => {
    // Once per route.
    const data = await cache.get(url, { signal });
    return <Page url={url} data={data} />;
  };
};
```

The client needs the same data the server used, or hydration doesn't match. The plugin doesn't transfer data between them, so the tree has to carry it to the client.

### Waiting for Suspense

By default, pre-rendering waits until every Suspense boundary of the route resolves. To leave a route's pending boundaries for the browser, set `waitForSuspense: false` for one route or for all of them. The route list can also live in a JSON file:

```json
["/", "/contact", { "path": "/products", "waitForSuspense": false }]
```

```ts
pluginSsg({ prerender: './prerender.json', server: './ssg/server.tsx' });
```

### Time limits

Each route has `renderTimeout` to finish, 60 seconds by default. The time counts from the moment the plugin starts the route. It covers the render function and the wait for Suspense boundaries. Set it globally or per route. The factory has the global `renderTimeout` as its limit.

A route that runs out of time is a render problem, like a component that throws.

### Components that only run in the browser

To leave a single component for the browser, and not a whole route, call `use(browser())` from React inside a Suspense boundary:

```tsx
function Cart() {
  use(browser('reads localStorage'));
  return <p>{localStorage.getItem('cart')}</p>;
}
```

The HTML carries that boundary's fallback, and the browser renders `Cart`. The build logs which components it left for the browser, and on which routes:

```console
[rsbuild:ssg] "/products": Cart renders in the browser (reads localStorage)
```

This message is information. It never stops the build.

### When a route fails to render

A route fails to render when the render function throws or rejects, when a component throws, or when the route runs out of time. `severity` decides what the build does with these problems:

- `'error'`, the default, reports every problem and then fails the build.
- `'warn'` reports every problem and finishes the build. The reference lists what happens to the page in each case.

```ts
pluginSsg({
  prerender: ['/', '/contact'],
  server: './ssg/server.tsx',
  severity: 'warn',
});
```

A broken contract always fails the build, regardless of `severity`. Examples are an invalid route, a JSON file that doesn't match the schema, and a factory that throws.

### A custom hydration entry

To control hydration yourself, point `hydrate` at your own entry. Pre-rendered pages load it, and the plugin stops replacing `createRoot`:

```ts
pluginSsg({
  prerender: ['/', { path: '/blog', waitForSuspense: true }],
  server: './ssg/server.tsx',
  hydrate: './ssg/hydrate.tsx',
  waitForSuspense: false,
});
```

```tsx
// ssg/hydrate.tsx
import { hydrateRoot } from 'react-dom/client';
import { Router } from 'wouter';
import App from '../src/App';

hydrateRoot(
  document.getElementById('root')!,
  <Router>
    <App />
  </Router>,
  { onRecoverableError: reportToSentry },
);
```

This entry must build the same tree as the server.

### In development

`rsbuild dev` doesn't pre-render. The app runs as a single-page app, and every route arrives with an empty container. Render problems appear only in `rsbuild build`.

### In preview

`rsbuild preview` serves the build output the way the manifest describes. Each pre-rendered route serves its own file, and any other route serves `spa.html`.

## Reference-level explanation

### Options

```ts
export type PrerenderRoute =
  | string                                  // '/contact'
  | {
      path: string;
      waitForSuspense?: boolean;            // overrides the global value
      renderTimeout?: number;               // ms, overrides the global value
    };

export type SsgOptions = {
  prerender: string | PrerenderRoute[];     // path to a JSON file, or a literal array
  server: string;                           // module that exports factoryServerFunction
  hydrate?: string;                         // custom entry for pre-rendered pages
  waitForSuspense?: boolean;                // default true
  renderTimeout?: number;                   // ms, default 60000
  severity?: 'error' | 'warn';              // default 'error'
};

export function pluginSsg(options: SsgOptions): RsbuildPlugin;
```

For each route, `waitForSuspense` and `renderTimeout` take the route's value, then the global value, then the default.

`server` and `hydrate` accept a path to any module in the project. Rsbuild compiles them like any other module, with the project's aliases and CSS handling. They can't be inline functions in `rsbuild.config.ts`, because Rsbuild doesn't compile that file.

### Route list schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "@computerwwwizards/rsbuild-plugin-ssg/prerender.schema.json",
  "title": "Routes to pre-render",
  "type": "array",
  "minItems": 1,
  "items": {
    "oneOf": [
      { "$ref": "#/$defs/path" },
      {
        "type": "object",
        "additionalProperties": false,
        "required": ["path"],
        "properties": {
          "path": { "$ref": "#/$defs/path" },
          "waitForSuspense": { "type": "boolean" },
          "renderTimeout": { "type": "integer", "minimum": 1 }
        }
      }
    ]
  },
  "$defs": {
    "path": {
      "type": "string",
      "description": "Static absolute path, without a trailing slash except \"/\".",
      "pattern": "^/([^/:*?#]+(/[^/:*?#]+)*)?$"
    }
  }
}
```

JSON Schema can't require each path to be unique, because `uniqueItems` compares whole items. The plugin checks uniqueness when it reads the list.

### Factory contract

```ts
import type { ReactNode } from 'react';

export type RenderFunction = (
  url: string,
  options: { signal: AbortSignal },
) => Promise<ReactNode>;

export type FactoryServerFunction = (context: {
  routes: readonly string[];
  signal: AbortSignal;
}) => Promise<RenderFunction>;
```

- The server module has a named export, `factoryServerFunction`. The plugin doesn't read a default export.
- The plugin calls the factory once per build, before it renders any route. `routes` holds every path from `prerender`, in list order.
- For each route, the plugin calls the render function once, with a path from `prerender`, such as `/contact`.
- The render function always returns a promise. Declare it `async` even when it doesn't wait for anything.
- When time runs out, the plugin aborts the signal and stops waiting, whether or not the work stops.
- Data that the tree uses on the server reaches the client only inside the tree.
- The module doesn't call `prerender` from React and doesn't read `waitForSuspense`. The plugin handles both.

### Errors that always stop the build

Before it renders any route, the plugin runs these checks. It stops at the first failure, writes the message to stderr, and exits with a non-zero code. `severity` doesn't apply to them.

| Check | Fails when |
| --- | --- |
| Node.js | The version is lower than 24.22.2. |
| React | The installed `react-dom` is lower than 19.3. |
| `prerender` | The list is empty, or its JSON file doesn't exist or doesn't match the schema. |
| Routes | A path doesn't match the schema, or appears twice. |
| Output files | A route's file collides with another output file, such as `/spa` with `spa.html`, or a route with a file from `public`. |
| `source.entry` | It has more than one entry, so the plugin can't tell which page is the shell. |
| HTML template | HTML generation is off, or the template has no element with the `html.mountId` id. |
| `react-dom/client` alias | The project already aliases it, and `hydrate` isn't set. |
| Server module | It has no `factoryServerFunction` export. |
| Factory | It throws, rejects, resolves to something other than a function, or runs out of time. |

### Render problems and severity

| Problem | When it happens | Outcome with `'warn'` |
| --- | --- | --- |
| Render error | The render function throws or rejects, or a component outside any Suspense boundary throws. | The plugin doesn't write the page and leaves the route out of `prerendered`, so `spa.html` serves it. |
| Error inside Suspense | A component inside a Suspense boundary throws. React writes the boundary's fallback and retries the boundary in the browser. | The plugin writes the page with that fallback. |
| Render timeout | The route runs out of `renderTimeout`. | If the render function didn't resolve, the plugin doesn't write the page. If a Suspense boundary was still pending, the plugin writes the page with that boundary's fallback. |

With `'error'`, the plugin renders every route, reports all problems, and then exits with a non-zero code. Problems go to stderr in both modes. Components left for the browser with `browser()` go to stdout as information, in both modes.

Every message starts with `[rsbuild:ssg]`. A message about a route names the route next.

React documents its behavior for errors inside a Suspense boundary in [Recovering from errors outside the shell](https://react.dev/reference/react-dom/server/renderToPipeableStream#recovering-from-errors-outside-the-shell).

### The `createRoot` replacement

In `rsbuild build`, the plugin aliases `react-dom/client` in the web environment to its own module. That module exports the same API:

- `hydrateRoot` is React's, unchanged.
- `createRoot(container, options)` checks a marker that the plugin writes on the container. With the marker, it hydrates. Without it, it renders.
- The returned root implements `render` and `unmount`. It passes `onCaughtError`, `onUncaughtError`, `onRecoverableError`, and `identifierPrefix` through to React.

The marker is internal and isn't part of the API. With `hydrate` set, the plugin doesn't add the alias. Pre-rendered pages load the `hydrate` entry, and `spa.html` loads the client entry.

### Inherited Rsbuild configuration

| Rsbuild setting | What it decides |
| --- | --- |
| `source.entry` | The client entry. It must have exactly one entry. `spa.html` loads it, and so do pre-rendered pages unless the project sets `hydrate`. |
| `output.distPath` | Where the plugin writes the HTML files and the manifest. |
| `html.mountId` | The id of the container that receives the pre-rendered HTML. The default is `root`. |
| `html.outputStructure` | The filename of each route. With `flat`, `/contact` is `contact.html`. With `nested`, it's `contact/index.html`. |
| `html.template` | The template for every page. |

### Output

- The plugin writes one HTML file per pre-rendered route, plus `spa.html`. The shell is always `spa.html`, so it never collides with the home page's `index.html`.
- `ssg-manifest.json` is a public, versioned contract, because deployment scripts read it. An incompatible change increments `version`.
- There is no `404.html`. Any route outside the list is client-side, so the plugin doesn't know the full route map. An unknown URL gets `spa.html` and the router's 404 view.
- The plugin renders up to 4 routes in parallel.
- The server bundle stays outside the output directory, so deploying the output never ships it.

### Preview server

In `rsbuild preview`, the plugin reads `ssg-manifest.json` and sets `server.historyApiFallback`. Each pre-rendered route gets a rewrite to its file, and `index` points to `spa.html`. If the project already sets `server.historyApiFallback`, the plugin leaves it unchanged and logs that preview may not follow the manifest.

### Minimum versions

| Dependency | Minimum | Declared as |
| --- | --- | --- |
| Node.js | 24.22.2, from the Long Term Support line 24 | `engines.node: ">=24.22.2"` |
| `@rsbuild/core` | 2.0 | peer dependency `^2.0.0` |
| `react`, `react-dom` | 19.3 | peer dependency `^19.3.0`, for `browser()` |

## Drawbacks

- **Soft 404.** An unknown URL answers 200 with the shell, and the router shows its 404 view. To a search engine, the response says that the URL exists. The plugin accepts this so projects don't have to list every route.
- **Tied to React 19's API.** The `createRoot` replacement mirrors `react-dom/client` from React 19. A major React release can change that API and break the plugin.
- **Hidden behavior.** On a pre-rendered page, the developer's `createRoot` hydrates instead of rendering. Someone who debugs without the docs may not expect it. Setting `hydrate` turns the replacement off.
- **Two trees to keep equal.** The tree lives in the server module and in the client entry. React reports a difference in the browser when the page hydrates. The plugin doesn't prevent it.
- **No data transfer.** Data that the server loads reaches the client only if the tree carries it.
- **Cancellation depends on the developer.** A timeout aborts the signal, and the plugin stops waiting. Work that ignores the signal keeps running until it settles.
- **React trees only.** The render function resolves to a React tree. A router that ships its loader data outside that tree doesn't fit, such as TanStack Router with loaders.
- **Problems appear only in the build.** `rsbuild dev` doesn't pre-render, so a render error first appears in `rsbuild build`.

## Rationale and alternatives

- **A synchronous render function.** It's simpler, but data that depends on the URL could only load in the factory. An async render function covers both cases.
- **A render function that returns a tree or a promise.** `await` handles both, so the plugin gains nothing. One return type is easier to document.
- **A timeout without a signal.** The plugin could stop waiting, but it couldn't stop the work. The signal lets the work stop too.
- **Severity per problem type.** A map such as `{ renderError: 'error', renderTimeout: 'warn' }` is more flexible, but every new problem type changes the public interface. One value covers new types without a change.
- **Configurable fatal errors.** With an invalid route or schema, the plugin doesn't know what to render, so a warning would only produce incomplete output.
- **A default export for the server module.** A named export makes the contract explicit, and the factory gives one-time setup a single place.
- **A required `hydrate` entry.** It's explicit, but it forces a file that usually repeats the client entry. It stays optional.
- **Classifying every route as pre-rendered or client-side.** It forces listing every route. With client-side as the default, the list names only what the plugin pre-renders.
- **`renderToString` instead of `prerender`.** `renderToString` doesn't wait for Suspense boundaries that load data. `prerender` waits, once, at build time.

## Prior art

- **React Router 8.** With `ssr: false` and a `prerender` list, it writes one HTML file per route and a fallback for the rest. The fallback is `__spa-fallback.html` when `/` is pre-rendered, and `index.html` otherwise. [Pre-rendering](https://reactrouter.com/how-to/pre-rendering)
- **Next.js.** `staticPageGenerationTimeout` defaults to 60 seconds. Next.js restarts page generation when it makes no progress, and fails the build when a page fails several attempts. This plugin uses the same default, without retries. [Static page generation timed out](https://nextjs.org/docs/messages/static-page-generation-timeout)
- **Preact.** `preact/compat` aliases `react` and `react-dom` to Preact, and React libraries keep working. This plugin uses the same technique for `react-dom/client`. [Switching to Preact](https://preactjs.com/guide/v10/switching-to-preact)
- **`browser()` in React 19.3.** A component that calls `use(browser())` leaves its Suspense fallback in the HTML and renders in the browser, without a hydration error. [browser](https://react.dev/reference/react-dom/browser)

## Unresolved questions

None.

## Future possibilities

- **Build-time hydration check.** Hydrate each pre-rendered page in a simulated DOM during the build, so mismatches show up before deploying.
- **Router presets.** A preset for Wouter or React Router could replace the server module.
- **Pre-rendering in development.** Rendering the listed routes on each dev-server request would show render problems earlier.
- **Dynamic routes.** Accept route parameters and a function that lists the values to pre-render.
- **Retries.** Retry a route that runs out of time, as Next.js does.
