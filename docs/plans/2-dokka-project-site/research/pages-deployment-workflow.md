# How `pages.yml` publishes `website/` and the branding assets to GitHub Pages

Facts apply as of 2026-11-21 unless a date or version is given.

## 1. Question

How should `.github/workflows/pages.yml` publish `website/` (site root), `docs/brand/` (under `assets/brand/`) and a `.nojekyll` file to GitHub Pages through GitHub Actions, on each push to `main` and by hand, so that:

- `contents: read` is the workflow default;
- only the deploying job holds `pages: write` and `id-token: write`;
- the assembling job holds neither;
- a later run never cancels a deployment in progress;
- no Java, Gradle or Dokka step is present?

## 2. Bottom line

- **Two jobs, `build` then `deploy`.** `build` inherits `contents: read` only: it checks out, assembles `_site/` and uploads the Pages artifact. `deploy` carries `permissions: { pages: write, id-token: write }` and the `github-pages` environment, and runs `actions/deploy-pages`. This is the layout the `deploy-pages` README recommends ("a dedicated job"). The single-job starter workflow cannot satisfy the permission requirement, because its one job holds the deploy scopes while it checks out and copies files.
- **Omit `actions/configure-pages`.** Its only effects for a hand-written site are a `GET /pages` call (which a `contents: read` build job cannot make), three unused URL outputs and the `GITHUB_PAGES` environment variable. It does not enable Pages with the default token. Nothing the site needs is lost.
- **`.nojekyll` is created in the assembly step (`touch _site/.nojekyll`) and `include-hidden-files: true` is passed to `actions/upload-pages-artifact@v5`.** Without that input the action's `tar --exclude=.[^/]*` silently drops the file (default `false` since v4). A live check on the organization's WebTools site confirms the arrangement ships the file: `https://spartanlaboratories.github.io/WebTools/.nojekyll` returns 200. A site deployed from an Actions artifact does not appear to run Jekyll at all, so the file is a safeguard (and a standard requirement), not a functional need. That claim rests on documentation by implication plus one contributor's test (see Gaps).
- **Concurrency: workflow-level `group: pages`, `cancel-in-progress: false`, no `queue` key.** At most one run is in progress and one is pending. A newer queued run replaces (cancels) the pending one, never the in-progress one, so a burst of pushes A, B, C deploys A then C and the site ends on the newest commit. The newer `queue: max` option (GitHub.com, 2026) would deploy every intermediate commit in order. It is wasteful for a static page and not wanted.
- **Permissions on the deploy job: exactly `pages: write` and `id-token: write`.** `actions/deploy-pages` reaches the artifact through the runner's artifact service, not through `GITHUB_TOKEN`, so neither `actions: read` nor `contents: read` is needed. A job-level `permissions:` block replaces the workflow default for that job, and every scope it does not list becomes `none`. The artifact upload in the build job needs nothing beyond `contents: read`, which `actions/checkout` uses.
- **Versions (all are the latest releases on 2026-11-21): `actions/checkout` v7.0.1, `actions/upload-pages-artifact` v5.0.0, `actions/deploy-pages` v5.0.1.** Pin by full commit SHA with a version comment, as GitHub's hardening guide recommends. The organization's other two Pages workflows use major tags (`@v7`, `@v5`, `@v5` in WebTools); tags are an acceptable alternative. This is an open point (section 7).
- **404 page:** GitHub serves the artifact's root `404.html` for any missing path under `/SpartanUI/` (confirmed live on the WebTools project site). Links inside it must be absolute from the host root (`/SpartanUI/...`), because the same file is returned at any depth. It must not link to `/api/`.

## 3. Findings

Tiers: **consensus** (standard, official docs or maintainer statement), **practice** (what mature projects visibly do), **opinion** (single source).

### 3.1 Current versions of the actions (consensus, official releases)

| Action | Latest | Published | Runtime | Source |
|---|---|---|---|---|
| `actions/checkout` | v7.0.1 (v7.0.0 on 2026-06-18) | 2026-07-20 | node24 | [releases](https://github.com/actions/checkout/releases/tag/v7.0.1) |
| `actions/upload-pages-artifact` | v5.0.0 | 2026-04-10 | composite (wraps `actions/upload-artifact` v7.0.0 by SHA) | [releases](https://github.com/actions/upload-pages-artifact/releases/tag/v5.0.0) |
| `actions/deploy-pages` | v5.0.1 (v5.0.0 on 2026-03-25) | 2026-09-01 | node24 | [releases](https://github.com/actions/deploy-pages/releases/tag/v5.0.1) |
| `actions/configure-pages` | v6.0.0 | 2026-03-25 | node24 | [releases](https://github.com/actions/configure-pages/releases/tag/v6.0.0) |

The versions stated in WebTools' plan (`checkout@v7`, `upload-pages-artifact@v5`, `deploy-pages@v5`) are confirmed as the current majors. Each major tag resolves to the same commit as its latest release today (read through `git/ref/tags` in each action's own repository):

- `checkout` v7 and v7.0.1: `3d3c42e5aac5ba805825da76410c181273ba90b1`
- `upload-pages-artifact` v5 and v5.0.0: `fc324d3547104276b827a68afc52ff2a11cc49c9`
- `deploy-pages` v5 and v5.0.1: `368f82528645a54fb793d4d04e342629a3f51346`

### 3.2 Hidden files and the artifact (consensus; checked against `action.yml`)

- `actions/upload-pages-artifact` v4.0.0 (2025-08-14) made the artifact exclude dotfiles. Its release note calls this a "potentially breaking change" and tells users who need dotfiles to build their own artifact. The claim in the brief is correct.
- v5.0.0 (2026-04-10) added the input `include-hidden-files` (default `"false"`): "Include hidden files and directories (those starting with a dot) in the artifact. Excludes .git and .github regardless." The archive step is `tar ... --exclude=.git --exclude=.github ${{ inputs.include-hidden-files != 'true' && '--exclude=.[^/]*' || '' }}`, so without the input every top-level and nested dotfile is dropped, with no warning. [action.yml at v5.0.0](https://github.com/actions/upload-pages-artifact/blob/v5.0.0/action.yml). The input was requested in [issue #129](https://github.com/actions/upload-pages-artifact/issues/129) (closed 2026-04-10).
- The input is blunt: it includes every dotfile under the path (for example a stray `.DS_Store`), not only `.nojekyll`. `_site/` is built fresh from two known directories, so this is contained.
- The artifact must be a single gzip of a single tar with no symlinks or hard links. The action passes `--dereference --hard-dereference`, so links in the sources are resolved rather than rejected.

### 3.3 Jekyll and `.nojekyll` for an Actions-deployed site

- GitHub Docs describe Jekyll running for sites published from a source branch ("If you publish your site from a source branch, GitHub Pages will use Jekyll to build your site by default") and tell users to disable it with `.nojekyll` at the root of the publishing source. [Creating a GitHub Pages site](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site) (consensus for the branch case).
- For the Actions source, the docs describe the artifact as the thing deployed; Jekyll appears only when a workflow runs `actions/jekyll-build-pages` itself. No official page says in so many words that an artifact deployment skips Jekyll. A contributor to `upload-pages-artifact` reports a test showing that "publishing via an artifact never invokes Jekyll" and that `.nojekyll` is only needed for the `gh-pages` branch ([issue #129 comment thread](https://github.com/actions/upload-pages-artifact/issues/129)). That is **one source's observation** (tier: opinion), and the organization's WebTools workflow says the same in a comment. Conclusion: `.nojekyll` changes nothing today for an artifact deployment, costs one `touch`, and is required by the organization's standard because Dokka output (added later) contains underscore-prefixed names.
- **Where to create it.** Creating it in the assembly step is preferred over committing `website/.nojekyll`. The standard says "The workflow adds a `.nojekyll` file", WebTools and MyGameTools do the same (`touch _site/.nojekyll`), and a committed empty dotfile is easy to delete by accident or to miss in a review. A committed copy would also need the same `include-hidden-files: true`, so nothing is saved.

### 3.4 `actions/configure-pages` (consensus; read from v6.0.0 source)

- `action.yml` description: "A GitHub Action to enable Pages, extract various metadata about a site, and configure some supported static site generators." Supported generators: nuxt, next, gatsby, sveltekit.
- `src/index.js` calls `findOrCreatePagesSite`, which first calls `octokit.rest.repos.getPages` (`GET /repos/{owner}/{repo}/pages`). Next it optionally rewrites generator config, writes the outputs `base_url`, `origin`, `host`, `base_path`, and exports `GITHUB_PAGES=true`.
- If that call fails and `enablement` is false (the default), it logs "Get Pages site failed. Please verify that the repository has Pages enabled..." and fails the step. `enablement: true` "requires a token other than `GITHUB_TOKEN`".
- For a static site with no generator and no use of the outputs, the call adds nothing except a way to fail. Because the build job holds only `contents: read`, the `getPages` call would run with the `pages` scope at `none` (every unlisted scope becomes `none`, see 3.6). The REST page for this endpoint states no fine-grained permission, so whether it would succeed is **unconfirmed**; omitting the action removes the question. The starter workflow `static.yml` includes it only because its single job already holds `pages: write`.
- Nothing is lost: the deploy job's `page_url` output comes from `deploy-pages`, not from `configure-pages`.

### 3.5 What `actions/deploy-pages` needs (consensus README plus source)

- README (v5.0.1): "The job that executes the deployment must at minimum have the following permissions: `pages: write`, `id-token: write`." Its own example is a dedicated `deploy` job with `needs: build`, exactly those two scopes, `environment: github-pages` and `url: ${{ steps.deployment.outputs.page_url }}`. [README](https://github.com/actions/deploy-pages/blob/v5.0.1/README.md).
- The source shows why no more is needed. `getArtifactMetadata` uses `DefaultArtifactClient().listArtifacts()` (the runner's artifact service, authenticated by the runner runtime token). The only `GITHUB_TOKEN` calls are `POST /repos/{owner}/{repo}/pages/deployments`, `GET .../pages/deployments/{id}` and the cancel call, which need `pages: write`. `core.getIDToken()` needs `id-token: write` (the failure text is "Ensure GITHUB_TOKEN has permission "id-token: write""). No repository contents are read, and the action does not check out. [`api-client.js` at v5.0.1](https://github.com/actions/deploy-pages/blob/v5.0.1/src/internal/api-client.js).
- GitHub's own docs example for a custom-workflow deploy job lists `contents: read` as well; the action's README and source do not require it. For a public repository `contents: read` adds no privilege either way. See Gaps for the one thing not confirmed on a live run.
- Default `timeout` is 10 minutes (`600000` ms): a stuck deployment is cancelled by the action itself.
- `deploy-pages` requires Pages to be enabled for the repository with the Actions source beforehand. The brief states this is done before merge.

### 3.6 Permissions semantics (consensus, GitHub Docs)

- "If you specify the access for any of these permissions, all of those that are not specified are set to `none`." This holds for a job-level block. A job-level `permissions:` block therefore replaces the workflow-level one for that job: it does not add to it. [Permissions reference, github/docs source](https://github.com/github/docs/blob/main/data/reusables/actions/github-token-available-permissions.md). A deploy job that lists only `pages: write` and `id-token: write` has `contents: none`, `actions: none`, and so on.
- A job that declares no `permissions:` block inherits the workflow-level block: `build` gets `contents: read`.
- `actions/upload-pages-artifact` (and the `actions/upload-artifact` v7.0.0 it calls) has no token input; artifact upload does not use `GITHUB_TOKEN` scopes. `actions/checkout` reads the repository with `contents: read`.

### 3.7 Concurrency (consensus; GitHub Docs at the commit of 2026-05-06, live page)

- At most one run per group is in progress. A run queued while another is in progress is `pending`. "By default, any existing `pending` job or workflow in the same concurrency group will be canceled and the new queued job or workflow will take its place." In-progress runs are only cancelled with `cancel-in-progress: true`. [Source](https://github.com/github/docs/blob/main/data/reusables/actions/actions-group-concurrency.md), rendered at [Control the concurrency of workflows and jobs](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency).
- So with A running, B queued (pending) and C queued: B is cancelled and C is pending. A finishes, C runs. The first pending run is replaced, as the brief asked.
- Newer option: `queue: single` (default) or `queue: max`. "`max`: Up to 100 jobs or workflow runs can be `pending` in the concurrency group. When the queue is full, any additional jobs or workflow runs are canceled." `queue: max` plus `cancel-in-progress: true` is a validation error. The docs gate this on the `actions-nga` feature, versioned for `fpt` and `ghec` (GitHub.com), added to the docs 2026-03-23 and the queueing text updated 2026-05-06. Not available on GHES.
- Ordering note from the docs: runs "are processed in first-in-first-out (FIFO) order according to the time each one started waiting on the concurrency group, not the time each workflow was dispatched... ordering is not guaranteed."
- GitHub's starter workflow states the intent: "Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued. However, do NOT cancel in-progress runs as we want to allow these production deployments to complete." ([`pages/static.yml`](https://github.com/actions/starter-workflows/blob/main/pages/static.yml)).
- **Guarantee check.** With `cancel-in-progress: false` and the default `queue: single`, an in-progress deployment is never cancelled by a later run. The pending slot always holds the most recently queued run, which checks out the newest pushed commit, so the last deployment to run is the newest. Three caveats:
  1. The docs say ordering by wait start is "not guaranteed" for near-simultaneous runs.
  2. Re-running an old run from the Actions tab re-deploys that run's commit and can put the site on an older commit than `main`. Normal pushes cannot.
  3. A failed newest run leaves the previous deployment in place.
  None of these is a cancellation of an in-progress deployment.
- Scope: concurrency group names are per repository and case-insensitive. A workflow-level `concurrency` also serialises the `build` job of the next run behind the current run, which is fine for a site this small.

### 3.8 The `github-pages` environment (consensus docs; practice from the live WebTools repository)

- Docs: "If your repository does not already include an environment called `github-pages`, the environment will be created automatically. We recommend that you add a deployment protection rule so that only the default branch can deploy to this environment." [Configuring a publishing source](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site).
- Observed on `SpartanLaboratories/WebTools` (read-only API, 2026-11-21): environment `github-pages`, created 2026-11-09, `deployment_branch_policy: { protected_branches: false, custom_branch_policies: true }`, one branch policy named `master`, which is that repository's default branch. A deployment from `main` in SpartanUI therefore passes, and a deployment from any other ref is rejected by the environment. This also means a `workflow_dispatch` started on another branch cannot publish. `SpartanLaboratories/SpartanUI` has no environments and no Pages site yet (API: `has_pages: false`, `environments: []`).
- `deploy-pages` README: "If your Pages site is using GitHub Actions as the source... we highly recommend you also protect your environment (we will configure it by default for you)."
- The `environment:` block must name `github-pages` (README: using a different name "is not recommended").

### 3.9 Custom 404 on a project site (consensus docs; confirmed live)

- Docs: add `404.html` (or `404.md` with `permalink: /404.html`) at the root of the publishing source. [Creating a custom 404 page](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-custom-404-page-for-your-github-pages-site).
- Checked on 2026-11-21 with a read-only request: `GET https://spartanlaboratories.github.io/WebTools/nope/deep/path` returns HTTP 404 with the body of WebTools' `website/404.html` (byte-for-byte the repository file). A missing path under the project prefix gets the project's root `404.html`, at any depth.
- Because one file answers every depth, relative links (`assets/styles.css`) resolve against the missing URL and break. WebTools' `404.html` uses `/WebTools/assets/...` and `/WebTools/` throughout. SpartanUI's must use `/SpartanUI/...`. A path outside the project prefix (`https://spartanlaboratories.github.io/anything`) is not served by this repository.
- WebTools' `404.html` has a button to `/WebTools/api/`. SpartanUI's page must not link `/api/`, because there is no `/api/` until Kotlin source exists.

### 3.10 Pinning (consensus guidance; practice differs)

- GitHub: "Pin references to actions to a commit SHA in your workflow files... they would need to generate a SHA-1 collision for a valid Git object payload." "When selecting a SHA, you should verify it is from the action's repository and not a repository fork." Tag pinning is acceptable "only if you trust the creator", with the warning that "a tag can be moved or deleted if a bad actor gains access to the repository storing the action." [Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use), source [secure-use.md](https://github.com/github/docs/blob/main/content/actions/reference/security/secure-use.md).
- The same page now describes workflow dependency lockfiles (`.github/workflows/actions.lock`, generated by `gh actions-lock`) as "the strongest protection available", superseding SHA pinning, on GitHub.com plans (`actions-workflow-lockfiles`, not scheduled for GHES). I did not establish whether it is generally available; it is not needed here.
- Trade-off: SHA pins are reproducible and cannot be repointed, but they do not receive patch releases (for example the Node 24 runtime moves, or `deploy-pages` v5.0.1 polling fixes) unless a tool such as Dependabot's `github-actions` ecosystem bumps them (it updates the SHA and the trailing version comment). Major tags receive patches automatically at the cost of trusting a moving ref. All three actions are first-party (`actions/*`), which narrows the risk. `upload-pages-artifact` itself pins its inner `upload-artifact` by SHA.
- The organization's two Pages workflows use major tags (WebTools `@v7`/`@v5`/`@v5`; MyGameTools older `@v4`/`@v3`/`@v4`).

## 4. How comparable projects do it

- **GitHub starter workflow `pages/static.yml`** (`actions/starter-workflows`, main): one `deploy` job holding `contents: read, pages: write, id-token: write` at workflow level, `concurrency: { group: "pages", cancel-in-progress: false }`, steps `checkout`, `configure-pages`, `upload-pages-artifact` (path `.`), `deploy-pages`. Simple, but the deploy scopes sit on the job that handles repository content. [file](https://github.com/actions/starter-workflows/blob/main/pages/static.yml).
- **GitHub Docs, "Linking separate build and deploy jobs"** and the `deploy-pages` README: `build` (checkout, configure-pages, build, upload) and `deploy` (`needs: build`, `environment: github-pages`, deploy). The README's deploy job lists only `pages: write` and `id-token: write`. This is the shape recommended here.
- **WebTools `.github/workflows/pages.yml`** (read-only): two jobs, `group: pages` with `cancel-in-progress: false`, `actions/checkout@v7`, `upload-pages-artifact@v5` with `include-hidden-files: true` and the comment "since v4 the artifact drops dotfiles; .nojekyll must ship", `deploy-pages@v5`, no `configure-pages`. Assembly is `cp -R website/. _site/`, `rm -f _site/README.md`, `cp -R docs/brand/. _site/assets/brand/`, `touch _site/.nojekyll`. The permissions sit at workflow level (`pages: write` and `id-token: write` on the build job too), which the standard forbids: that is the part to avoid.
- **MyGameTools `.github/workflows/pages.yml`**: same shape with `checkout@v4`, `upload-pages-artifact@v3` (no hidden-files input, so v3 still ships the dotfile) and `deploy-pages@v4`. Bumping `upload-pages-artifact` to v4 or later without the new input would silently drop `.nojekyll`.
- Live proof for the assembly: WebTools' deployed site serves `/WebTools/.nojekyll` (HTTP 200, `application/octet-stream`) and `/WebTools/404.html` content at missing paths.

## 5. Pitfalls and regrets

- **Silent dotfile loss.** `upload-pages-artifact` v4 and later drop dotfiles without a warning. Anyone bumping a workflow that relied on v3 behaviour loses `.nojekyll` (and `.well-known/`) with no error. Pass `include-hidden-files: true` and keep the comment saying why. Issue #129 shows maintainers and users disliked the first blunt default.
- **Permissions that look inherited.** Listing any scope in a job-level block sets every other scope to `none` for that job. Adding `contents: read` back is only needed if a step in that job reads contents.
- **`configure-pages` in a least-privilege build job** fails or needs extra scopes for a call whose results are unused.
- **`cancel-in-progress: true` on Pages** can cancel a deployment mid-flight; the site then stays on the previous commit even though the newest push is green. Keep `false`. Skipped (cancelled) pending runs show as cancelled in the Actions list; that is expected, not a failure.
- **`queue: max`** looks safer ("never skip a commit") but deploys every intermediate commit in sequence and cannot be combined with `cancel-in-progress`. For a static site the newest commit is the only one that matters.
- **Re-running an old run** re-deploys its commit. Treat "Re-run" on stale runs as a way to roll the site back.
- **Missing `needs: build`** makes the deploy job search for an artifact that does not exist yet (GitHub Docs).
- **Relative links in `404.html`** break at any depth other than the root; the file is served at the missing URL, not redirected.
- **A holding page that links `/api/`** gives a 404. The assembly has no `api/` directory; the 404 page and the landing page must carry no `/api/` link.
- **The first run cannot be tested before merge.** The trigger is `push` to `main`, and `workflow_dispatch` only receives events when the workflow file is on the default branch. Pages must already be set to the Actions source or `deploy-pages` fails at the first push.
- **SHA pins go stale** without an update tool; stale pins are a maintenance cost, not a defect.

## 6. Recommendation for this repository

Ship two jobs in `.github/workflows/pages.yml`. `build` holds the inherited `contents: read`, runs checkout and one assembly script, then uploads the artifact with `include-hidden-files: true`. `deploy` holds `pages: write` and `id-token: write` only and runs `actions/deploy-pages`. Concurrency is one workflow-level group with `cancel-in-progress: false`. Pin the three actions by SHA with a version comment. Omit `configure-pages`.

```yaml
name: Pages

# Publishes the project site: website/ at the site root and docs/brand/ under assets/brand/.
# Needs the repository's Pages source set to GitHub Actions (Settings -> Pages -> Source, or
#   gh api -X POST repos/SpartanLaboratories/SpartanUI/pages -f build_type=workflow).

on:
  push:
    branches: [main]
  workflow_dispatch:

# Default for every job. The deploy job replaces it with its own block.
permissions:
  contents: read

# One deployment at a time. A run queued behind a running one waits as the single pending run and a
# newer one replaces it, so the newest commit is always the last to deploy. An in-progress
# deployment is never cancelled: a cancelled deploy leaves the site on the previous commit.
concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    name: Build site
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false

      - name: Assemble _site
        run: |
          set -euo pipefail
          rm -rf _site
          mkdir -p _site
          cp -R website/. _site/
          rm -f _site/README.md   # notes for maintainers, not part of the published site

          # Brand assets live once in docs/brand; the pages reference them under assets/brand/.
          mkdir -p _site/assets/brand
          cp -R docs/brand/. _site/assets/brand/

          # The organization's standard requires this file. A deployment from an Actions artifact
          # runs no Jekyll, so today it is a safeguard; it matters once /api/ ships names that
          # start with an underscore.
          touch _site/.nojekyll

          if [ ! -f _site/index.html ]; then
            echo "::error::website/index.html is missing, so the site would have no home page"
            exit 1
          fi

      - uses: actions/upload-pages-artifact@fc324d3547104276b827a68afc52ff2a11cc49c9 # v5.0.0
        with:
          path: _site
          include-hidden-files: true   # since v4 the artifact drops dotfiles; .nojekyll must ship

  deploy:
    name: Deploy to GitHub Pages
    needs: build
    runs-on: ubuntu-latest
    permissions:
      pages: write      # to create the Pages deployment
      id-token: write   # to prove to Pages that the deployment comes from this workflow
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@368f82528645a54fb793d4d04e342629a3f51346 # v5.0.1
```

Line-by-line reasons:

- `on: push: branches: [main]` and `workflow_dispatch`: the standard's trigger and the manual run. No path filter, so every push to `main` redeploys.
- Workflow-level `permissions: contents: read`: the default is read-only. `build` inherits it; checkout needs it. It is explicit so a repository or organization default of read-write tokens cannot widen it.
- `concurrency` at workflow level, group `pages`, `cancel-in-progress: false`, no `queue`: the default `single` queue gives one running plus one pending, with the newest replacing the pending run (3.7).
- `build` is the only job that touches repository content; it has neither Pages nor OIDC scope.
- `persist-credentials: false`: nothing in `build` pushes, so no token is left in the git config; `.git` is excluded from the artifact in any case. Optional hardening; deleting it does not change behaviour.
- `cp -R website/. _site/`: the `/.` form copies dotfiles too. `rm -f _site/README.md` is harmless if the file does not exist and keeps maintainer notes out of the site. `set -e` makes a missing `docs/brand/` fail the build.
- `touch _site/.nojekyll` then `include-hidden-files: true`: the file is created at publish time and told to ship (3.2, 3.3).
- The `index.html` check turns a missing home page into a build failure instead of a 404 site. There is deliberately no check for `api/`.
- `upload-pages-artifact`: `path: _site` is explicit although it is the default (`_site/`). The artifact name defaults to `github-pages`, which `deploy-pages` expects; retention is one day.
- `deploy`: `needs: build`; a job-level `permissions` block with the two scopes, replacing the default, so `contents`, `actions` and the rest are `none` for this job; `environment: github-pages` with the page URL output, as the README shows.
- `configure-pages` omitted (3.4). No `if:` guard on the deploy job: the `github-pages` environment only accepts the `main` branch, so a manual run from another branch fails at the environment and publishes nothing (3.8).

What the `404.html` in `website/` must do:

- Sit at `website/404.html`, so it is copied to the artifact root and served for every missing path under `/SpartanUI/`.
- Use only root-absolute URLs that include the project prefix: `/SpartanUI/`, `/SpartanUI/assets/...`. Do not use relative links, and do not rely on `<base>` being applied by every client.
- Link to the repository and the landing page (`/SpartanUI/`) only. Do not link `/SpartanUI/api/`; no API reference exists until Kotlin source lands.
- Carry `<meta name="robots" content="noindex">`, as WebTools' page does, so the 404 body is not indexed.

## 7. Open points

- **Pinning style: SHA (recommended) or major tags.** The organization's existing workflows use major tags. GitHub's guidance is SHA. SHA pins need an update mechanism to stay current; a `github-actions` entry in a `dependabot.yml` is the usual one and is outside this issue's file list. If no update tool is added, major tags (`@v7`, `@v5`, `@v5`) are an equally valid, organization-consistent choice. Either meets every stated requirement.
- **`contents: read` on the deploy job.** The recommendation omits it (README and source need only the two scopes). If the first run on `main` fails with a permission error on the deploy job, adding `contents: read` to that block is the documented fallback and adds no privilege on a public repository.

## 8. Load-bearing claims

1. **`upload-pages-artifact` v5.0.0 has an `include-hidden-files` input defaulting to false, and without it the archive excludes dotfiles.** Quote: "Include hidden files and directories (those starting with a dot) in the artifact. Excludes .git and .github regardless." Source: [action.yml at v5.0.0](https://github.com/actions/upload-pages-artifact/blob/v5.0.0/action.yml), release 2026-04-10.
2. **The dotfile exclusion began at v4.0.0.** Quote: "hidden files (specifically dotfiles) will not be included in the artifact". Source: [release v4.0.0](https://github.com/actions/upload-pages-artifact/releases/tag/v4.0.0), 2025-08-14.
3. **Current majors and SHAs:** `actions/checkout` v7.0.1 (2026-07-20), `actions/upload-pages-artifact` v5.0.0 (2026-04-10), `actions/deploy-pages` v5.0.1 (2026-09-01); `configure-pages` v6.0.0 (2026-03-25). Each is the "latest" release via the GitHub releases API; the SHAs in section 3.1 are the commits the tags point to (read through `gh api repos/actions/<name>/git/ref/tags/<tag>`, 2026-11-21). Source: [checkout](https://github.com/actions/checkout/releases/tag/v7.0.1), [upload-pages-artifact](https://github.com/actions/upload-pages-artifact/releases/tag/v5.0.0), [deploy-pages](https://github.com/actions/deploy-pages/releases/tag/v5.0.1).
4. **`deploy-pages` recommends a dedicated deploy job and needs only `pages: write` and `id-token: write`.** Quotes: "We recommend this action to be used in a dedicated job" and "The job that executes the deployment must at minimum have the following permissions". Source: [README at v5.0.1](https://github.com/actions/deploy-pages/blob/v5.0.1/README.md).
5. **`deploy-pages` finds the artifact through the runner's artifact service, not the repository API.** Quote (code): `const twirpResponse = await artifactClient.listArtifacts()`. Source: [`src/internal/api-client.js` at v5.0.1](https://github.com/actions/deploy-pages/blob/v5.0.1/src/internal/api-client.js).
6. **The OIDC token needs `id-token: write`.** Quote (code): "Ensure GITHUB_TOKEN has permission "id-token: write"." Source: [`src/index.js` at v5.0.1](https://github.com/actions/deploy-pages/blob/v5.0.1/src/index.js).
7. **`configure-pages` calls `GET /pages` and needs a non-default token to enable Pages.** Quotes: "Get Pages site failed. Please verify that the repository has Pages enabled" and "This option requires a token other than `GITHUB_TOKEN` to be provided." Source: [`src/api-client.js`](https://github.com/actions/configure-pages/blob/v6.0.0/src/api-client.js) and [`action.yml`](https://github.com/actions/configure-pages/blob/v6.0.0/action.yml) at v6.0.0.
8. **A job-level permissions block sets unlisted scopes to `none`.** Quote: "all of those that are not specified are set to `none`". Source: [GitHub Docs, permissions reference](https://github.com/github/docs/blob/main/data/reusables/actions/github-token-available-permissions.md) (rendered under [Use GITHUB_TOKEN for authentication](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token)), read 2026-11-21.
9. **A pending run is replaced by a newer queued run; running jobs are only cancelled with `cancel-in-progress: true`.** Quote: "any existing `pending` job or workflow in the same concurrency group will be canceled". Source: [concurrency reusable](https://github.com/github/docs/blob/main/data/reusables/actions/actions-group-concurrency.md), last changed 2026-05-06, and the [live page](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency).
10. **`queue: max` exists on GitHub.com and cannot be combined with `cancel-in-progress: true`.** Quotes: "Up to 100 jobs or workflow runs can be `pending` in the concurrency group." and "is not allowed and will result in a workflow validation error". Source: same file; feature `actions-nga` is versioned `fpt: '*'` in [`data/features/actions-nga.yml`](https://github.com/github/docs/blob/main/data/features/actions-nga.yml).
11. **Ordering is FIFO by wait time but not guaranteed.** Quote: "ordering is not guaranteed". Source: same concurrency reusable, 2026-05-06.
12. **GitHub's starter workflow states the Pages concurrency intent.** Quote: "skipping runs queued between the run in-progress and latest queued". Source: [`pages/static.yml`](https://github.com/actions/starter-workflows/blob/main/pages/static.yml), main, read 2026-11-21.
13. **The `github-pages` environment is created automatically and GitHub recommends restricting it to the default branch.** Quote: "the environment will be created automatically". Source: [Configuring a publishing source](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site), read 2026-11-21. Observed default policy: `GET /repos/SpartanLaboratories/WebTools/environments/github-pages/deployment-branch-policies` returned one branch policy `master` (default branch), 2026-11-21.
14. **Pin to a full SHA, with the stated trade-offs.** Quotes: "Pin references to actions to a commit SHA in your workflow files." and "a tag can be moved or deleted if a bad actor gains access". Source: [Secure use reference](https://github.com/github/docs/blob/main/content/actions/reference/security/secure-use.md), read 2026-11-21.
15. **Jekyll is used for branch-published sites.** Quote: "will use Jekyll to build your site by default". Source: [Creating a GitHub Pages site](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site), read 2026-11-21. That an Actions artifact runs no Jekyll is **only partly confirmed**: see claim 16.
16. **Artifact deployment does not invoke Jekyll (one source, unconfirmed by official docs).** Quote: "publishing via an artifact never invokes Jekyll". Source: [actions/upload-pages-artifact issue #129](https://github.com/actions/upload-pages-artifact/issues/129), comment by a contributor, 2025-09.
17. **A project site serves the artifact's root `404.html` for missing paths at any depth, and `.nojekyll` published from the artifact is reachable.** No quote (observation): `curl` against `https://spartanlaboratories.github.io/WebTools/nope/deep/path` returned status 404 and the repository's `website/404.html`; `.../WebTools/.nojekyll` returned 200. Docs for the file name: "type `404.html` or `404.md`" ([Creating a custom 404 page](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-custom-404-page-for-your-github-pages-site)). Observation date 2026-11-21. Unconfirmed in official docs that absolute links are *required* (the reason is how browsers resolve relative URLs at the missing path).
18. **`workflow_dispatch` only fires for a workflow file on the default branch.** Quote: "This trigger only receives events when the workflow file is on the default branch." Source: [Workflow syntax reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax), read 2026-11-21.
19. **`actions/checkout` v7.0.1 runs on node24 and has `persist-credentials` defaulting to true.** Source: [`action.yml` at v7.0.1](https://github.com/actions/checkout/blob/v7.0.1/action.yml) (`using: node24`, `persist-credentials: default: true`).

## 9. Gaps

- **Deploy job without `contents: read` is not confirmed on a live run.** The README and source support it; no live run in this organization uses it (WebTools and MyGameTools hold all three scopes at workflow level). The fallback is in section 7.
- **`GET /pages` permission for `GITHUB_TOKEN`** is not stated in GitHub's REST page, so I cannot say whether `configure-pages` would have worked in a `contents: read` job. Irrelevant once it is omitted.
- **No official statement that an Actions-artifact deployment skips Jekyll**; only a contributor's test and the documentation's silence on the Actions case. The recommendation does not depend on it (the file ships either way).
- **When GitHub creates the `github-pages` environment and its branch policy** (at the moment Pages is switched to Actions or at the first deployment) is not stated in the docs. The WebTools policy was observed after the fact. After enabling Pages on SpartanUI, listing the environment's `deployment-branch-policies` would confirm a `main` rule before merge.
- **Workflow dependency lockfiles:** the docs call them the strongest protection, but whether they are generally available on GitHub.com is not established here; they are not part of the recommendation.
- **Concurrency ordering under near-simultaneous pushes** relies on documented, explicitly non-guaranteed FIFO behaviour; I did not test it.
- The repository has no `.github/` directory today, so no existing workflow, Dependabot configuration or branch-protection setting in SpartanUI was read.
