# JavaScript test toolchain for the project site's `site.js`

## Question

Which test runner and DOM emulator should run the component tests for `website/assets/site.js` (a load-time IIFE with a theme toggle, reading `localStorage` and `window.matchMedia`), how do the tests load the shipped file unchanged into a fresh document per test, how do they stub `localStorage` (including storage that throws on read and on write) and `matchMedia`, and how does CI install and run them?

Requirements an answer meets: the shipped file runs unchanged; one fresh emulated document per test; the five behaviours each a separate test; `npm ci` from a committed lockfile; Node 24 (the development machine has 24.19.0, npm 11.17.0) and the CI-pinned version; as few dependencies as possible; licences compatible with Apache-2.0; current maintained versions as of 2026-11-22; a CI job with `contents: read` on push and pull request. The folder the tests sit in is open and written `<test-dir>` below.

## Bottom line

- Use Node's built-in test runner (`node --test`, `node:test`, `node:assert/strict`) with **jsdom 31.0.1** as the only dependency. The lockfile has 28 packages; Vitest + jsdom has 95, Vitest + happy-dom 76, Jest + jest-environment-jsdom 350.
- Load the shipped script with `new JSDOM(<real index.html>, { runScripts: "outside-only" })` and `window.eval(<site.js source>)`. The page's own script tag is not executed, so `site.js` runs exactly once, after the stubs are installed. `site.js` needs no change.
- jsdom does **not** implement `window.matchMedia` (verified: `typeof window.matchMedia === "undefined"` on jsdom 31.0.1), so a plain function stub is both required and sufficient. jsdom's own `localStorage` throws a `SecurityError` on the default `about:blank` origin; the tests sidestep it by replacing `window.localStorage` with `Object.defineProperty` (the property is an own, configurable accessor) by a fake whose `getItem` / `setItem` throw on demand.
- Tests find files by quoted glob: `node --test "<test-dir>/**/*.test.js"`. A bare directory argument does **not** work on Node 24.19.0 (it fails with `Cannot find module`).
- An exception in a click listener never reaches the caller of `click()`; jsdom reports it as an `error` event on the window. The "throwing storage" tests must collect those events and assert none, otherwise they pass against a `site.js` that has no try/catch (verified by mutation, see Findings 9).
- CI: `actions/checkout@v7`, `actions/setup-node@v7` with `node-version: ${{ matrix.node }}` over `[24, 26]` (24 is the development line, now Maintenance LTS; 26 is Active LTS since 2026-10-28), `npm ci`, `npm test`, `permissions: contents: read`, concurrency group per ref.

## Findings

Evidence tiers: **consensus** = standard/spec or several independent credible sources; **practice** = what mature projects visibly do; **opinion** = one source.

1. **Node's test runner is Stable on Node 24.** The Node 24 documentation marks the test runner `Stability: 2 - Stable`. It runs each file in its own process (so no globals leak between files), supports `test(name, (t) => …)` with `t.after`, and ships `node:assert/strict`. Tier: consensus (official docs). Source: https://nodejs.org/docs/latest-v24.x/api/test.html (text at https://github.com/nodejs/node/blob/v24.x/doc/api/test.md). Applies to Node 24.x (24.22.3 released 2026-11-10).
2. **Node release status on 2026-11-22.** From the release schedule: v24 "Krypton" LTS 2025-10-28, Maintenance from 2026-10-20, end of life 2028-04-30; v26 "Lithium" LTS 2026-10-28 (Active LTS on 2026-11-22), Maintenance 2027-10-20, end of life 2029-04-30; v22 end of life 2027-04-30. Latest releases: 24.22.3 and 26.13.1 (both 2026-11-10, both in the `actions/node-versions` manifest used by `setup-node`). Tier: consensus. Source: https://github.com/nodejs/Release/blob/main/schedule.json, https://nodejs.org/dist/index.json.
3. **`node --test` discovery.** With no arguments Node runs files matching `**/*.test.{cjs,mjs,js}`, `**/*-test.…`, `**/*_test.…`, `**/test-*.…`, `**/test.…` and `**/test/**/*.…` (plus the `.ts` forms). With arguments, they are glob patterns, "enclosed in double quotes on the command line to prevent shell expansion". Verified on 24.19.0: `node --test "test/component/**/*.test.js"` and `node --test "test/**/*.test.js"` ran 9 tests; `node --test test/component` (a directory) failed with `Error: Cannot find module '…\test\component'`; a single file path works from any working directory. Double-quoted globs inside an npm script run on Windows (cmd) and POSIX alike (run on Windows with npm 11.17.0). Tier: consensus (docs) plus own trial. Source: https://github.com/nodejs/node/blob/v24.x/doc/api/test.md ("Running tests from the command line").
4. **jsdom 31.0.1** (published 2026-11-13, MIT) declares `engines.node: ^22.22.2 || ^24.15.0 || >=26.0.0`; Node 24.19.0 satisfies it. It installs 28 packages in total with no install scripts and a single optional peer (`@napi-rs/canvas`). Licences across the 28: MIT 22, BSD-2-Clause 2, BSD-3-Clause 1, ISC 1, Apache-2.0 1, BlueOak-1.0.0 1 (`lru-cache`); all are compatible with an Apache-2.0 repository and none is distributed with it (dev-only). Tier: consensus (registry metadata, lockfile of a scratch install). Source: `npm view jsdom@31.0.1`, https://github.com/jsdom/jsdom/releases/tag/v31.0.1.
5. **jsdom `runScripts` and `eval`.** `runScripts: "outside-only"` installs fresh copies of the JavaScript globals on `window`, including `window.eval`, "which allows running scripts, but with the jsdom `window` as the global". Without `runScripts`, `window.eval` is Node's own `eval` and "will not run scripts in a useful way". `"dangerously"` additionally executes `<script>` elements and inline handlers in the page. For the page at hand, `"dangerously"` would run `site.js` during parsing only if the script tag's file were fetched, which needs `resources: "usable"` and a file URL; and it would run before the stubs can be installed. Tier: consensus (official README). Source: https://github.com/jsdom/jsdom#executing-scripts, v31.0.1.
6. **jsdom `localStorage` needs a non-opaque origin.** On the default `about:blank` URL `window.localStorage` throws `SecurityError` (verified, jsdom 31.0.1). With `url: "https://…"` it returns a working `Storage`. `localStorage` is an own, configurable accessor on the window (`{get, set: undefined, enumerable: true, configurable: true}`), so `Object.defineProperty(window, "localStorage", { value: fake, configurable: true })` replaces it. The README documents `url` as setting `window.location` and `document.URL`, and `storageQuota` for the storage areas; the origin rule itself is the HTML specification's and was observed, not quoted, from the README. Tier: practice (observed) and README. Source: https://github.com/jsdom/jsdom#readme.
7. **jsdom does not implement `matchMedia`.** `typeof new JSDOM("").window.matchMedia` is `"undefined"` (verified, 31.0.1). The jsdom repository's web-platform-test expectations list `matchMedia.html: [fail, Unknown]` and `matchMedia-display-none-iframe.html: [fail, Not implemented]`. The README says limitations can be worked around with `Object.defineProperty()` on the window. Tier: practice (observed, plus the project's own test-expectation file). Source: https://github.com/jsdom/jsdom/blob/main/test/web-platform-tests/to-run.yaml.
8. **Alternatives cost far more for this job.** Counts are lockfile entries from scratch `npm install --package-lock-only` runs on 2026-11-23:

   | Option | Versions | Packages | Notes |
   |---|---|---|---|
   | `node:test` + jsdom | jsdom 31.0.1 | 28 | no config file, no transform step |
   | Vitest + jsdom | vitest 5.1.1, jsdom 31.0.1 | 95 | Vite stack; 12 MPL-2.0 packages (`lightningcss` and its platform builds); `fsevents` install script |
   | Vitest + happy-dom | vitest 5.1.1, happy-dom 21.0.4 | 76 | happy-dom has a long run of security advisories (several rated high or critical, the latest dated 2026-10-20), because it evaluates page script inside the host process |
   | Jest + jsdom | jest 30.5.2, jest-environment-jsdom 30.5.2 | 350 | its jsdom environment depends on `jsdom ^26.1.0`, not 31; npm 11.17 prints `allow-scripts` warnings during install |

   Vitest 5.1.1 requires Node `^22.12.0 || ^24.0.0 || >=26.0.0`; Jest 30.5.2 `^18.14.0 || … || >=24.0.0`; happy-dom 21.0.4 `>=20.0.0`. All four are MIT and actively released (Vitest 5.1.1 on 2026-11-17). Tier: practice (registry metadata). Sources: `npm view <package>`, https://github.com/capricorn86/happy-dom/security (advisory list, read with `gh api`).
9. **A thrown exception in a listener is invisible to the test unless collected (verified by mutation).** With the shipped `site.js`, all nine trial tests pass. With the `try/catch` around `setItem` removed, a test that only asserts the theme still passes (the attribute is set before the write), so the test must also collect window `error` events. After adding that collection, removing the `setItem` guard fails the "throws on write" and "throws on both" tests; removing the `getItem` guard fails the "throws on read" and "throws on both" tests; deleting the `storeTheme(next)` call fails the "stores each choice" test; changing `(prefers-color-scheme: light)` to `dark` fails the "light system preference" and "throws on both" tests. Tier: own trial. (Trial run on Node 24.19.0, jsdom 31.0.1, in a scratch folder outside the repository.)
10. **`actions/setup-node` is at v7 (latest 7.1.0, 2026-10-08).** Inputs include `node-version`, `node-version-file` (accepts `.nvmrc`, `.node-version`, `.tool-versions`, `mise.toml`, `package.json`; reads `devEngines.runtime`, then `engines.node`), `cache` (`npm`, `yarn`, `pnpm`), `cache-dependency-path` and `package-manager-cache`. It runs on `node24`. `node-version` wins if both version inputs are given. `actions/checkout` latest is v7.0.1. Tier: consensus (official docs and `action.yml`). Source: https://github.com/actions/setup-node/blob/main/action.yml, https://github.com/actions/setup-node/blob/main/docs/advanced-usage.md.
11. **Ecosystem conventions for the `package.json`.** `private: true` prevents accidental publication; `engines` documents the supported Node range (npm only warns unless `engine-strict` is set); `type: "module"` makes `.js` files ES modules, so tests use `import` and `import.meta.dirname` (Node 20.11+). Scripts named per test set (`test:component`) with `test` running all of them is the shape the organization's named test sets call for. Tier: practice.

## How comparable projects do it

- **Node's own pattern for DOM tests (no framework):** create `new JSDOM(html, { runScripts: "outside-only", url })` per test, install stubs on `dom.window`, `dom.window.eval(source)`, assert on `dom.window.document`, call `dom.window.close()` afterwards. This is the documented jsdom flow (README, "Executing scripts") and what the trial implements.
- **Organization CI style** (the WebTools repository's [`.github/workflows/ci.yml`](https://github.com/SpartanLaboratories/WebTools/blob/master/.github/workflows/ci.yml) and [`pages.yml`](https://github.com/SpartanLaboratories/WebTools/blob/master/.github/workflows/pages.yml), and this repository's `.github/workflows/pages.yml`): top-level `permissions: contents: read`; `concurrency` group `ci-${{ github.ref }}` with `cancel-in-progress` true only for pull requests; actions pinned by major tag (`actions/checkout@v7`); `persist-credentials: false` on checkout where nothing pushes; triggers `push` and `pull_request` to the default branch plus `workflow_dispatch`. The new `ci.yml` copies that shape with `main` as the branch.
- **Vitest / Jest practice** for the same job is `environment: "jsdom"` with `vi.stubGlobal` / `jest.spyOn(window, …)` and a script loaded by `import` or `require`; both need a module wrapper or a side-effect import for a load-time IIFE, and a fresh module registry per test (`vi.resetModules()` / `jest.isolateModules`) to re-run it. Evaluating the source in a new `JSDOM` window per test needs neither, so the module-cache problem does not exist.

## Pitfalls and regrets

- **Directory argument to `node --test`.** It fails on Node 24.19.0 (Finding 3). Always pass a quoted glob or rely on default discovery; do not pass a bare folder.
- **Unquoted globs.** A POSIX shell without `globstar` expands `**` as `*`, and cmd.exe does not expand at all; inside an npm script, write `\"…\"` so Node expands the pattern identically everywhere.
- **Opaque origin.** If a test ever uses jsdom's real `localStorage`, it needs `url: "https://…"` on the constructor. The recommended tests do not use it.
- **Silent listener exceptions** (Finding 9): assert on collected `error` events.
- **Using globals instead of a window.** Assigning `globalThis.document` / `globalThis.localStorage` and importing `site.js` leaks state between tests and caches the module; `window.eval` inside a per-test `JSDOM` has no shared state.
- **`"dangerously"`** executes arbitrary page script in a context that can reach the host in some configurations; `"outside-only"` runs only what the test hands it.
- **Stubbing too little of the contract.** The `matchMedia` stub compares the query string exactly, so a typo in the shipped query fails a test instead of passing by accident (mutation 4 in Finding 9).
- **Version ranges.** `node-version-file: package.json` against an `engines` range such as `^24.15.0 || >=26.0.0` resolves to the newest matching Node (26), not to 24; an explicit `node-version` matrix avoids the ambiguity.
- **Cache and supply chain.** `setup-node` documents that npm caching is enabled automatically only when `package.json` declares npm as the package manager, and warns that a poisoned cache is a risk for privileged workflows. This job has `contents: read` and no secrets; with 28 packages an uncached `npm ci` takes seconds, so the job leaves `cache` off.

## Recommendation for this repository

Use `node:test` + jsdom 31.0.1. The job is nine small DOM assertions over one script; the built-in runner needs no config file, no transform and no second dependency tree, and jsdom provides the window, `eval` and DOM events. Vitest, happy-dom and Jest are rejected because they add 50 to 320 more packages (and, for Vitest, MPL-2.0 packages and a native-build optional dependency) for features this job does not use; happy-dom is also rejected for its security-advisory history on a tool that evaluates page script.

Node version: develop on 24 (24.19.0 satisfies jsdom's `^24.15.0`), and run CI on a matrix of **24** and **26**. 24 is the line the development machine uses and is in Maintenance LTS until 2028-04-30; 26 is the Active LTS line on 2026-11-22. Both are accepted by jsdom 31 (`engines`) and by the `setup-node` manifest. Both are pinned by major (`24`, `26`); `setup-node` resolves each to its newest release.

Placement: `<test-dir>` is free. Edit the glob in the `test:component` script and the `".."` depth constant in the test file together. The test file reads the real `website/index.html`, so a renamed `.theme-toggle` class or a changed default `data-theme` fails the suite.

`site.js` needs **no change**: it runs unmodified through `window.eval`, and every behaviour is reachable through the stubs.

### `package.json` (repository root)

```json
{
  "name": "spartanui-site-tests",
  "private": true,
  "type": "module",
  "description": "Component tests for the SpartanUI project site script.",
  "license": "Apache-2.0",
  "engines": {
    "node": "^24.15.0 || >=26.0.0"
  },
  "scripts": {
    "test": "npm run test:component",
    "test:component": "node --test \"<test-dir>/**/*.test.js\""
  },
  "devDependencies": {
    "jsdom": "31.0.1"
  }
}
```

Run `npm install` once (Node 24 or 26) to create `package-lock.json`, commit both. The trial lockfile had `lockfileVersion: 3` and 28 packages. `npm ci` from it succeeded on Node 24.19.0 and npm 11.17.0. `test` runs every set through `npm run`; add `test:integration` etc. to it when those sets exist.

### Test file `<test-dir>/site.test.js`

The `REPO_ROOT` line assumes `<test-dir>` is two folders below the root (`test/component`); change the number of `".."` for another depth. This exact file, with `<test-dir>` = `test/component`, passed 9 of 9 on Node 24.19.0 with jsdom 31.0.1.

```js
// Component tests for website/assets/site.js: the shipped script runs unchanged in a fresh jsdom
// document per test, with localStorage and matchMedia replaced by stubs. Nothing here touches the
// network or the real browser storage.
import assert from "node:assert/strict";
import { readFileSync } from "node:fs";
import { resolve } from "node:path";
import { test } from "node:test";
import { JSDOM } from "jsdom";

// This file sits two folders below the repository root (<test-dir> = test/component); adjust the
// number of ".." if the tests move deeper or shallower.
const REPO_ROOT = resolve(import.meta.dirname, "..", "..");
const PAGE_HTML = readFileSync(resolve(REPO_ROOT, "website/index.html"), "utf8");
const SITE_JS = readFileSync(resolve(REPO_ROOT, "website/assets/site.js"), "utf8");

const STORE_KEY = "spartanui-theme";
const LIGHT_QUERY = "(prefers-color-scheme: light)";

/** An in-memory Storage that records writes; `failRead` / `failWrite` make it throw like a blocked store. */
function fakeStorage({ initial = {}, failRead = false, failWrite = false } = {}) {
  const data = new Map(Object.entries(initial));
  const writes = [];
  return {
    writes,
    getItem(key) {
      if (failRead) throw new DOMException("storage blocked", "SecurityError");
      return data.has(key) ? data.get(key) : null;
    },
    setItem(key, value) {
      if (failWrite) throw new DOMException("quota exceeded", "QuotaExceededError");
      writes.push([key, value]);
      data.set(key, String(value));
    },
  };
}

/**
 * Loads index.html into a new jsdom window, installs the stubs, then runs the shipped site.js in that
 * window. Scripts in the page are not executed (runScripts "outside-only"), so site.js runs once, here.
 * `prefersLight: undefined` leaves window.matchMedia undefined, as in jsdom itself.
 */
function loadSite(t, { storage = fakeStorage(), prefersLight } = {}) {
  const dom = new JSDOM(PAGE_HTML, { runScripts: "outside-only" });
  const { window } = dom;
  t.after(() => window.close());

  Object.defineProperty(window, "localStorage", { value: storage, configurable: true });
  if (prefersLight !== undefined) {
    window.matchMedia = (query) => ({ media: query, matches: query === LIGHT_QUERY && prefersLight });
  }

  // An exception inside an event listener never reaches the caller of click(); jsdom reports it as an
  // "error" event on the window, so collect those to let a test assert that none happened.
  const errors = [];
  window.addEventListener("error", (event) => errors.push(event.error ?? event.message));

  window.eval(SITE_JS);

  const html = window.document.documentElement;
  const toggle = window.document.querySelector(".theme-toggle");
  return { window, storage, toggle, errors, theme: () => html.getAttribute("data-theme") };
}

test("a stored light choice wins over a dark system preference", (t) => {
  const site = loadSite(t, { storage: fakeStorage({ initial: { [STORE_KEY]: "light" } }), prefersLight: false });
  assert.equal(site.theme(), "light");
});

test("a stored dark choice wins over a light system preference", (t) => {
  const site = loadSite(t, { storage: fakeStorage({ initial: { [STORE_KEY]: "dark" } }), prefersLight: true });
  assert.equal(site.theme(), "dark");
});

test("a light system preference on a first visit gives light", (t) => {
  const site = loadSite(t, { prefersLight: true });
  assert.equal(site.theme(), "light");
});

test("no stored choice and no light preference keeps the dark default", (t) => {
  assert.equal(loadSite(t, { prefersLight: false }).theme(), "dark");
});

test("no stored choice and no matchMedia keeps the dark default", (t) => {
  assert.equal(loadSite(t).theme(), "dark");
});

test("the toggle flips the theme and stores each choice", (t) => {
  const site = loadSite(t, { prefersLight: false });
  assert.ok(site.toggle, "the page has a .theme-toggle button");

  site.toggle.click();
  assert.equal(site.theme(), "light");
  site.toggle.click();
  assert.equal(site.theme(), "dark");
  assert.deepEqual(site.storage.writes, [[STORE_KEY, "light"], [STORE_KEY, "dark"]]);
});

test("storage that throws on read still gives a working toggle", (t) => {
  const site = loadSite(t, { storage: fakeStorage({ failRead: true }), prefersLight: false });
  assert.equal(site.theme(), "dark");
  site.toggle.click();
  assert.equal(site.theme(), "light");
  assert.deepEqual(site.errors, []);
});

test("storage that throws on write still gives a working toggle", (t) => {
  const site = loadSite(t, { storage: fakeStorage({ failWrite: true }), prefersLight: false });
  site.toggle.click();
  assert.equal(site.theme(), "light");
  site.toggle.click();
  assert.equal(site.theme(), "dark");
  assert.deepEqual(site.errors, []);
});

test("storage that throws on both read and write still gives a working toggle", (t) => {
  const site = loadSite(t, { storage: fakeStorage({ failRead: true, failWrite: true }), prefersLight: true });
  assert.equal(site.theme(), "light");
  site.toggle.click();
  assert.equal(site.theme(), "dark");
  assert.deepEqual(site.errors, []);
});
```

How each runner finds tests at an arbitrary `<test-dir>`:

| Runner | Finds tests by | Verified |
|---|---|---|
| `node --test` (Node 24) | quoted glob arguments (`"<test-dir>/**/*.test.js"`), or default discovery patterns when none are given (`**/*.test.*`, `**/test/**`, …); a bare directory fails | yes, 24.19.0 |
| Vitest | `test.include` globs in a config file (default `**/*.{test,spec}.?(c|m)[jt]s?(x)`); CLI arguments are substring filters, not paths | not run |
| Jest | `testMatch` / `roots` in config; CLI arguments are regular expressions on the path | not run |

### `.github/workflows/ci.yml`

```yaml
name: CI

# Runs the component tests for the project site script (website/assets/site.js). The Kotlin build
# and its tests get their own jobs when the Gradle build lands.

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  # A new push to a pull request replaces the run of its old head; a run on main is left to finish.
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

jobs:
  site-tests:
    name: Site script tests (Node ${{ matrix.node }})
    runs-on: ubuntu-latest
    timeout-minutes: 10
    strategy:
      fail-fast: false
      matrix:
        # 24 is the line developed on (Maintenance LTS); 26 is the Active LTS line.
        node: [24, 26]
    steps:
      - uses: actions/checkout@v7
        with:
          persist-credentials: false   # nothing in this job pushes
      - uses: actions/setup-node@v7
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
```

If the single-line job name is needed as a required status check, the checks are named `Site script tests (Node 24)` and `Site script tests (Node 26)`.

### `.gitignore` lines to add

```
### Node tooling (site tests) ###
node_modules/
```

`node --test` writes no output files; `*.log` (npm debug logs) is already ignored. Add `coverage/` only if coverage is enabled later.

## Load-bearing claims

1. Node 24's test runner is Stable. Quote: "Stability: 2 - Stable". Source: https://github.com/nodejs/node/blob/v24.x/doc/api/test.md (the file's opening stability line), Node 24.x.
2. Glob arguments to `node --test` are quoted. Quote: "The glob patterns should be enclosed in double quotes on the command line to prevent shell expansion". Same source, section "Running tests from the command line", Node 24.x. (The directory-argument failure is own trial on 24.19.0, not a documented statement.)
3. Default discovery includes `**/test/**/*.{cjs,mjs,js}`. Quote: "`**/*.test.{cjs,mjs,js}`" and "`**/test/**/*.{cjs,mjs,js}`". Same source.
4. Node release status. Quote (schedule.json, v24): `"lts": "2025-10-28", "maintenance": "2026-10-20", "end": "2028-04-30"`; (v26): `"lts": "2026-10-28", "maintenance": "2027-10-20", "end": "2029-04-30"`. Source: https://raw.githubusercontent.com/nodejs/Release/main/schedule.json, read 2026-11-22.
5. jsdom 31.0.1 supports Node 24.19.0. Quote (package metadata): `"node": "^22.22.2 || ^24.15.0 || >=26.0.0"`. Source: `npm view jsdom@31.0.1 engines.node`, https://www.npmjs.com/package/jsdom, version 31.0.1 published 2026-11-13; MIT licence from the same metadata.
6. `runScripts: "outside-only"` provides `window.eval` with the window as global. Quote: "`window.eval`, which allows running scripts, but with the jsdom `window` as the global". Source: https://github.com/jsdom/jsdom#executing-scripts (README, `main`, v31.0.1).
7. Without `runScripts`, `window.eval` is useless. Quote: "`window.eval` will not run scripts in a useful way". Same README.
8. jsdom's `url` option defaults to `about:blank`. Quote: "It defaults to `\"about:blank\"`". Same README. (That `localStorage` throws `SecurityError` there is own trial, not README text: unconfirmed from a primary document.)
9. jsdom does not implement `matchMedia`. Quote (jsdom test expectations): `matchMedia.html: [fail, Unknown]` and `matchMedia-display-none-iframe.html: [fail, Not implemented]`. Source: https://github.com/jsdom/jsdom/blob/main/test/web-platform-tests/to-run.yaml. The absence of the function is also own trial (`typeof window.matchMedia` is `"undefined"`); the README is silent on `matchMedia`.
10. The README endorses overriding unimplemented window members. Quote: "using `Object.defineProperty()` to change what various layout-related getters and methods return". Same README. (Applying this to `localStorage` is own trial.)
11. `actions/setup-node` v7 inputs. Quote (`action.yml`): "`node-version-file`: File containing the version Spec of the version to use." and "`cache`: … Supported values: npm, yarn, pnpm." Source: https://github.com/actions/setup-node/blob/main/action.yml; latest release v7.1.0, 2026-10-08.
12. `package-manager-cache` default behaviour. Quote: "caching is enabled when either devEngines.packageManager or the top-level packageManager field in package.json specifies npm". Same `action.yml`.
13. `node-version` wins over `node-version-file`. Quote: "If both the `node-version` and the `node-version-file` inputs are provided then the `node-version` input is used." Source: https://github.com/actions/setup-node/blob/main/docs/advanced-usage.md.
14. `actions/checkout` latest major is 7. Source: `gh api repos/actions/checkout/releases/latest` returned tag `v7.0.1`, published 2026-07-20 for v7.0.1; no quote available (API field).
15. Package counts and licences for the options (28 / 95 / 76 / 350 packages; Vitest 5.1.1 and Jest 30.5.2 MIT; the Jest environment's `jsdom ^26.1.0`). Source: `npm view` and `npm install --package-lock-only` in scratch folders on 2026-11-23; own measurement, no quotable primary sentence.
16. happy-dom advisories. Source: `gh api repos/capricorn86/happy-dom/security-advisories`; includes GHSA-37j7-fg3j-429f, title "Escape of VM Context gives access to process level functionality" (critical, 2025-10-09) and GHSA-834v-8h66-3h79 (high, 2026-10-20).

## Gaps

- Not run: Vitest, happy-dom and Jest trials; their discovery rules in the table are from knowledge of their configuration, not re-read from current docs (the package versions and counts are measured).
- Not run on Node 26 or on GitHub-hosted runners; Node 26 support rests on jsdom's `engines` field and the `setup-node` manifest listing 26.13.1. The first CI run on Node 26 is the real check.
- The `SecurityError` on `about:blank` is observed on jsdom 31.0.1, not quoted from a jsdom document.
- The statement that `npm ci` and the lockfile are committed is a repository action; the lockfile has to be generated by running `npm install` once. Whether a root `package.json` and `node_modules/` collide with the Kotlin Multiplatform Gradle build planned for the root (for example, a Kotlin/JS target resolving its own npm tree) was not checked; Kotlin's JS tooling keeps its own `build/` directory, but that is not verified here.
- Optional hardening not included: a coverage report, and a lint for the test file. Neither is required by the settled scope.
