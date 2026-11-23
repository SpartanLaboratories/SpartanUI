# Research: the SpartanUI project site

This folder holds the research behind the plan for
[SpartanLaboratories/SpartanUI#2](https://github.com/SpartanLaboratories/SpartanUI/issues/2): a project site
deployed to GitHub Pages, the organization's branding, and the tests of the site's script. The organization's
standard is [`documentation/dokka.md`](https://github.com/SpartanLaboratories/.github/blob/main/documentation/dokka.md)
in `SpartanLaboratories/.github`.

The rest of the design (the landing page, the branding files, the repository's own documents) rests on facts
about this repository and about the organization's earlier adoptions of the standard, which the plan states with
their sources. Three questions need evidence from outside the organization: the Pages workflow, the test toolchain
for the site's script, and the accessibility check of the site's pages.

## Questions

### 1. How should the workflow deploy a static site with no build step to GitHub Pages?

How should `.github/workflows/pages.yml` publish `website/` and the branding assets to GitHub Pages through
GitHub Actions, so that only the deploying job holds the Pages and OIDC permissions, the `.nojekyll` file reaches
the published site, and no run cancels a deployment in progress?

**Options worth comparing.**

- The job layout: a build job that checks out and assembles the site and uploads it as the Pages artifact,
  followed by a deploy job that deploys it; or a single job doing both, the shape of GitHub's starter workflow
  for static HTML.
- `actions/configure-pages`: run in the build job, or omitted.
- `.nojekyll`: created in the assembled directory at publish time, or committed under `website/`; and what the
  artifact upload needs for a file whose name starts with a dot to be published.
- Pinning the actions: by major version tag, as the organization's other Pages workflows do, or by full commit
  SHA.
- Concurrency: one group for every run with `cancel-in-progress: false`, or another arrangement; what happens to
  a run queued behind a deployment in progress, and to a second run queued behind that one.

**Requirements an answer must meet.**

- The workflow runs on each push to `main` and can be started by hand.
- `contents: read` is the workflow's default permission. `pages: write` and `id-token: write` are granted on the
  deploy job only; the job that assembles the site holds neither.
- The published site holds `website/` at its root, `docs/brand/` under `assets/brand/`, and a `.nojekyll` file.
- A later run never cancels a deployment in progress.
- The workflow has no Java, Gradle or Dokka step. The repository has no Kotlin source; the API reference at
  `/api/` and the step that generates it arrive with the first Kotlin source.
- The repository is public, its default branch is `main`, and its Pages source is to be GitHub Actions.
- The site is a project site, served under `/SpartanUI/`, with a custom `404.html`.

**Evidence that would settle it.**

- The current README, `action.yml` and release notes of `actions/checkout`, `actions/upload-pages-artifact`,
  `actions/deploy-pages` and `actions/configure-pages`: their current major versions, their inputs, and the
  permissions each states it needs.
- GitHub's documentation on custom workflows for Pages; on how `permissions` set on a job replace the workflow's
  defaults; on concurrency groups and `cancel-in-progress`; on the `github-pages` environment and its deployment
  branch rule; on custom 404 pages for a project site; on Jekyll and `.nojekyll` for a site deployed from an
  Actions artifact; and its security hardening guidance on pinning actions.

**Findings:** [`pages-deployment-workflow.md`](pages-deployment-workflow.md).

### 2. How should the repository test the site's script?

How should the repository run automated tests of `website/assets/site.js` in the component set — a
dependency-free script that runs once at page load, reads a stored theme from `localStorage`, falls back to
`prefers-color-scheme` through `matchMedia`, and toggles the theme on a click — with both browser APIs stubbed,
locally and in CI, using the smallest toolchain that stays maintained?

**Options worth comparing.**

- The test runner and DOM emulator: Node's built-in test runner (`node --test`) with jsdom; Vitest with jsdom;
  Vitest with happy-dom; Jest with jsdom.
- How the tests run the shipped script unchanged in the emulated page: jsdom's `runScripts` setting with
  `window.eval`, an injected `<script>` element, or a module import.
- How the tests stub `localStorage`, including storage that throws on read and on write, and `matchMedia`, which
  a DOM emulator may not implement.
- The Node version the toolchain and CI pin, and how CI installs and runs it (`actions/setup-node`, its npm cache,
  `npm ci` from a committed lockfile).

**Requirements an answer must meet.**

- The tests run the shipped `site.js` file, unchanged in behaviour, each in a fresh document.
- The stubs cover a stored `light` or `dark` choice, a light system preference on a first visit, no preference
  (the page's `dark` default stays), the click toggle that switches and stores the theme, and storage that throws
  on read and on write while the toggle still works.
- The toolchain installs reproducibly from a committed lockfile and runs on the Node version CI uses and on
  Node 24, the version on the development machine.
- It adds as few dependencies as the job needs, under licences compatible with the repository's Apache-2.0, at
  current, maintained versions.
- A CI job holding `contents: read` runs the tests on each push and pull request.
- The repository has no Node tooling today, and will later hold a Gradle build at its root.

**Evidence that would settle it.**

- The current releases, documentation and changelogs of Node's test runner, Vitest, Jest, jsdom and happy-dom:
  their stability, their DOM environments, jsdom's `runScripts` and its storage, the browser APIs each emulator
  implements.
- Node's release schedule on 2026-11-22: which line is Active LTS.
- `actions/setup-node`: its current major, its inputs for the Node version and the npm cache.
- Each package's licence and dependency count, from its package metadata.

**Findings:** [`js-test-toolchain.md`](js-test-toolchain.md).

### 3. How should the site's pages be checked for accessibility?

How should the change check the landing page and its 404 page for keyboard access and accessibility in both colour
themes — the skip link, the icon-only theme toggle named by `aria-label`, focus visibility, reduced motion and
colour contrast — automatically in the test toolchain, or by hand in a browser?

**Options worth comparing.**

- An automated audit with axe-core inside the DOM emulator.
- An automated audit in a real headless browser, such as Playwright with axe-core.
- A manual audit in a browser, with axe DevTools or Lighthouse, beside a keyboard-only pass.

**Requirements an answer must meet.**

- The check covers colour contrast and accessible names in both themes.
- It covers the skip link reaching the main content, and the theme toggle working by keyboard with a visible focus
  ring.
- It can be repeated on the live site after the first deployment.
- It adds no browser download to the repository's CI unless what it gains justifies one.

**Evidence that would settle it.**

- axe-core's documentation on the rules that need a rendering engine, colour contrast inside jsdom among them.
- The current axe-core, axe DevTools and Lighthouse releases, and what their accessibility audits cover.
- The WCAG 2.2 success criteria behind the checks: bypass blocks, keyboard, focus visible, contrast minimum, and
  name, role and value.

**Findings:** [`accessibility-audit.md`](accessibility-audit.md).

## Verification results

### The Pages workflow (question 1)

Every load-bearing claim of [`pages-deployment-workflow.md`](pages-deployment-workflow.md) (its section 8) was
re-checked against its source on 2026-11-21.

| # | Claim | Result |
|---|---|---|
| 1 | `actions/upload-pages-artifact` v5.0.0 has the input `include-hidden-files`, default `"false"`; without it the archive excludes dotfiles | Holds |
| 2 | The dotfile exclusion began at `upload-pages-artifact` v4.0.0 (2025-08-14) | Holds |
| 3 | The latest releases (`checkout` v7.0.1, `upload-pages-artifact` v5.0.0, `deploy-pages` v5.0.1, `configure-pages` v6.0.0), and the commits their major tags resolve to | Holds: each major tag is a lightweight tag on its latest release's commit |
| 4 | `actions/deploy-pages` recommends a dedicated job holding at minimum `pages: write` and `id-token: write` | Holds |
| 5 | `actions/deploy-pages` finds the artifact through the runner's artifact client | Holds: its other calls, creating, polling and cancelling the deployment, use the REST API with `pages: write` |
| 6 | The OIDC token needs `id-token: write` | Holds |
| 7 | `actions/configure-pages` reads `GET /repos/{owner}/{repo}/pages`; enabling Pages through it needs a token other than `GITHUB_TOKEN` | Holds |
| 8 | A `permissions` block sets every scope it does not list to `none`, at job level as at workflow level | Holds |
| 9 | A newer queued run cancels the pending one; a run in progress is cancelled only with `cancel-in-progress: true` | Holds |
| 10 | `queue: max` exists on GitHub.com and cannot be combined with `cancel-in-progress: true` | Holds |
| 11 | Pending runs are processed first in, first out by wait time, without a guaranteed order | Holds |
| 12 | GitHub's starter workflow `pages/static.yml` uses the group `pages` with `cancel-in-progress: false` | Holds |
| 13 | GitHub creates the `github-pages` environment automatically and recommends limiting it to the default branch | Holds |
| 14 | GitHub's secure-use reference recommends pinning actions to a commit SHA | Holds |
| 15 | GitHub Pages builds a site published from a source branch with Jekyll by default | Holds |
| 16 | A deployment from an Actions artifact never invokes Jekyll | Holds in part: the statement is an outside commenter's report from a brief test (issue #129 of `actions/upload-pages-artifact`, 2025-09-14), not a maintainer's, and no GitHub documentation states it |
| 17 | The custom 404 page is a file named `404.html` | Holds. That GitHub serves the artifact's root `404.html` for a missing path at any depth under a project prefix rests on the live check of the WebTools site below, not on the documentation's wording |
| 18 | `workflow_dispatch` receives events only when the workflow file is on the default branch | Holds |
| 19 | `actions/checkout` v7.0.1 runs on `node24`, and `persist-credentials` defaults to `true` | Holds |

GitHub's guide "Using custom workflows with GitHub Pages" carries one example `permissions` block, in its section on
deploying Pages artifacts: `contents: read`, `pages: write` and `id-token: write`, beside text that states a minimum of
`pages: write` and `id-token: write`. Its examples of linked build and deploy jobs carry no `permissions` block.

The observations of the WebTools repository were read again on 2026-11-21. Its `github-pages` environment holds one
deployment branch policy, `master`, its default branch. The environment was created on 2026-11-09 at 02:03:01 UTC, before
the repository's first Pages deployment at 02:53:17 UTC, so it exists before any deployment.
`https://spartanlaboratories.github.io/WebTools/.nojekyll` answers 200, and a missing path under `/WebTools/` answers 404
with the body of that repository's `website/404.html`.

### The site script's tests and the accessibility check (questions 2 and 3)

Every load-bearing claim of [`js-test-toolchain.md`](js-test-toolchain.md) (16) and
[`accessibility-audit.md`](accessibility-audit.md) (12) was re-checked against its source on 2026-11-22. All 28 hold.

| Claims | Result |
|---|---|
| Node 24's test runner is Stable; `node --test` takes quoted globs and discovers `**/*.test.js` and `**/test/**` by default | Holds |
| Node 26 is Active LTS from 2026-10-28 and Node 24 Maintenance LTS until 2028-04-30; the latest releases are 24.22.3 and 26.13.1 | Holds |
| jsdom 31.0.1 is the latest release (2026-11-13, MIT), accepting Node `^22.22.2 \|\| ^24.15.0 \|\| >=26.0.0`; with `runScripts: "outside-only"` its `window.eval` runs scripts with the jsdom window as the global; its `url` defaults to `about:blank`; it does not implement `matchMedia`; its README endorses `Object.defineProperty()` to change what the window returns | Holds. The README does not mention `matchMedia`; the project's own test expectations list it as not implemented |
| `actions/setup-node` v7.1.0 (2026-10-08) has the inputs `node-version`, `node-version-file`, `cache` and `package-manager-cache`, runs on `node24`, and uses `node-version` when both version inputs are given; `actions/checkout`'s latest release is v7.0.1 | Holds |
| Vitest 5.1.1, Jest 30.5.2 and happy-dom 21.0.4 are MIT; `jest-environment-jsdom` 30.5.2 depends on jsdom `^26.1.0`; happy-dom's advisories include critical and high escapes from its script context | Holds |
| axe-core's `color-contrast` rule does not work in jsdom, which does no layout or rendering | Holds |
| axe-core needs every WCAG tag named (`wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa`; there is no `wcag22a`), and an `incomplete` result needs further testing | Holds |
| axe-core 4.14.1 is the latest release, under MPL-2.0; jsDelivr serves it, and the file's SHA-384 is `xt1SFg49QaDwsyt0Xfs2XAyBtYPClXl3X2q5OOLeGAUmq9jZTYIMdztpF99WdhiH` | Holds |
| Lighthouse 13.6.0 runs axe-core 4.14.0 with the tags `wcag2a` and `wcag2aa` only, its score weighted by axe's impact assessments | Holds |
| Automated tools missed a large share of known barriers in GDS's 2017 test | Holds |
| Playwright's browsers take a few hundred megabytes, and it can drive an installed Chrome or Edge; the hosted Ubuntu 24.04 image carries Google Chrome 157 | Holds |
| The WCAG 2.2 wording for 2.4.1, 2.4.7, 1.4.11, 1.4.3 and 4.1.2 | Holds. The Understanding page for 1.4.11 discusses focus indicators and their contrast |
| A GitHub Pages project site in the organization sends no content-security policy | Holds for `https://spartanlaboratories.github.io/WebTools/` on 2026-11-23 |

## Summary

### The deploy workflow

- **Two jobs.** A `build` job checks out the repository, assembles `_site/` from `website/` and `docs/brand/`, adds
  `.nojekyll`, and uploads the Pages artifact. A `deploy` job, `needs: build`, deploys it to the `github-pages`
  environment with `actions/deploy-pages`, the layout the action's README recommends. GitHub's single-job starter
  workflow does not fit: its one job holds the Pages and OIDC scopes while it handles repository content, which the
  organization's standard rules out. ([findings, sections 2 and 4](pages-deployment-workflow.md#2-bottom-line); claims
  4 and 12)
- **Permissions.** `contents: read` at workflow level, which `build` inherits and its checkout needs; the artifact
  upload uses no token scope. `deploy` declares exactly `pages: write` and `id-token: write`. A job-level block sets
  every scope it does not list to `none`, and `actions/deploy-pages` reads the artifact through the runner's artifact
  service, so the job needs no `contents` or `actions` scope. GitHub's Pages guide shows a deploy job holding
  `contents: read` as well, beside text stating the same two-scope minimum as the action's README; adding
  `contents: read` to the deploy job is the fallback if a deployment ever fails on a permission. (claims 4 to 6 and 8;
  verification results)
- **`.nojekyll`.** Created by the assembly step and shipped by passing `include-hidden-files: true` to
  `actions/upload-pages-artifact@v5`: since v4 the artifact drops every dotfile, with no warning, unless told
  otherwise. Creating it at publish time keeps it out of the source tree, where an empty dotfile is easily lost; a
  committed copy would need the same input. Whether Jekyll runs at all for a site deployed from an artifact is not
  documented: GitHub documents Jekyll builds for sites published from a branch, and one outside report says an artifact
  deployment never runs it. The file costs one `touch`, and the standard requires it. (claims 1, 2, 15 and 16)
- **No `actions/configure-pages`.** For a hand-written site it only reads the Pages settings and sets outputs nothing
  uses, and that read would run in a job without the `pages` scope. (claim 7)
- **Concurrency.** One workflow-level group, `pages`, with `cancel-in-progress: false` and the default single pending
  slot. A deployment in progress is never cancelled; a newer run replaces a pending one, so the newest pushed commit is
  the last deployed. `queue: max` would deploy every intermediate commit in turn, which a static page has no use for.
  Pending runs are ordered by the time they started waiting, without a guarantee for near-simultaneous pushes.
  (claims 9 to 12)
- **Action versions and pinning.** The current majors are `actions/checkout@v7` (v7.0.1),
  `actions/upload-pages-artifact@v5` (v5.0.0) and `actions/deploy-pages@v5` (v5.0.1). GitHub's secure-use reference
  favours pinning by commit SHA. A SHA pin stays current only with an update tool such as Dependabot, which the
  repository does not have; the organization's other Pages workflows pin these same first-party actions by major tag,
  and the major tags are recommended here. (claims 3 and 14)
- **Triggers and the first run.** `push` to `main`, and `workflow_dispatch` for a run by hand. `workflow_dispatch`
  receives events only for a workflow file already on the default branch, so the first run is the push that merges
  the workflow; the Pages source must be GitHub Actions by then, or the deploy job fails. (claim 18)

### The Pages setting and its environment

- The Pages source is set to GitHub Actions (`build_type: workflow`) through `POST /repos/{owner}/{repo}/pages`, or
  in the repository's Pages settings. GitHub creates the `github-pages` environment itself and recommends limiting it
  to the default branch; on WebTools the environment holds one branch rule, its default branch, and existed before
  its first deployment. A deployment from `main` passes such a rule, and a manual run from another branch cannot
  publish. (claim 13; verification results)

### The 404 page

- GitHub serves the artifact's root `404.html` for a missing path at any depth under the project prefix, so every
  link and asset reference in it is absolute under `/SpartanUI/`; relative references would resolve against the
  missing path and break. It carries no link to `/api/`, which does not exist until the repository has Kotlin source.
  (claim 17; verification results)

### The site script's tests

- **Node's built-in test runner with jsdom.** `node --test`, Stable on Node 24, with jsdom 31.0.1 (MIT) as the one
  development dependency: 28 packages in the lockfile, against 76 to 350 for the Vitest and Jest setups, which bring
  features these tests do not use. happy-dom is also left out for its run of security advisories, in a tool that runs
  page script inside the host process. ([findings](js-test-toolchain.md#bottom-line), sections 1, 4 and 8)
- **The shipped script runs unchanged.** Each test loads the real `website/index.html` into a new jsdom window with
  `runScripts: "outside-only"`, installs its stubs, then runs the real `website/assets/site.js` with `window.eval`. The
  page's own script tag does not run, so the script runs once, after the stubs; `site.js` needs no change. (findings 5)
- **The stubs.** jsdom has no `matchMedia`, so a function that matches the exact query string stands in for it.
  jsdom's own `localStorage` throws on its default `about:blank` origin, so each test replaces it with a fake storage
  whose reads and writes can be made to throw. An exception in a click listener reaches no caller: jsdom reports it as
  an `error` event on the window, so the tests of throwing storage collect those events and assert there are none, or
  they would pass against a script with no `try`/`catch`. (findings 6, 7 and 9)
- **Discovery.** `node --test` takes quoted glob patterns; a bare directory argument fails on Node 24.19.0.
  (findings 3)
- **Node versions and CI.** Node 26 is the Active LTS line and Node 24 the Maintenance LTS line, and jsdom 31.0.1
  accepts both. CI runs the tests on both, with `actions/checkout@v7` and `actions/setup-node@v7` (`node-version` from a
  matrix), `npm ci` from the committed lockfile and no npm cache: an uncached install of 28 packages takes seconds, and
  a cache brings a supply-chain risk the job has no need of. (findings 2 and 10)

### The accessibility check

- **By hand in a real browser, not in CI.** axe-core does not evaluate colour contrast in jsdom, which does no layout
  or rendering. A browser job in CI would cost a download of a few hundred megabytes, or a server step with the
  runner's own Chrome, and still could not judge whether a focus ring can be seen. The audit therefore runs in a
  browser, with axe-core 4.14.1 injected from jsDelivr under its SHA-384 integrity value, beside a keyboard-only pass.
  Lighthouse adds nothing: it runs axe-core with the WCAG 2.0 A and AA tags only.
  ([findings](accessibility-audit.md#bottom-line), sections 1, 5, 6 and 8)
- **Two runs per theme.** A plain run with the tags `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa` and
  `best-practice` leaves the contrast of text over the header and page gradients and over the hero's glow
  `incomplete`. A second run, with each gradient replaced by its worst-case stop and the glow hidden, resolves them.
  An `incomplete` result is no pass. (findings 2 to 4)
- **What the page fails as it stands.** The audit and the keyboard pass find three failures on the current files:
  the light theme's `.eyebrow` (`#907040`) at 3.68:1 on the darkest page stop, under the 4.5:1 its small text needs;
  links inside running text, in the lede and the footer, told apart from the text around them by colour alone; and, in
  the light theme, a navy focus ring on the always-dark header, about 1.3:1. WCAG 1.4.11 asks 3:1 for a focus
  indicator against the colours next to it. (findings 7 and 9, and the findings the design has to resolve)
- **The live site.** The organization's WebTools project site sends no content-security policy, so a script injected
  from the CDN is expected to load on the live SpartanUI site as well; that rests on one site on one date.
  (verification results)
