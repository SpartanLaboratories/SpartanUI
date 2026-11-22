# Research: GitHub Pages deployment for the SpartanUI project site

This folder holds the research behind the plan for
[SpartanLaboratories/SpartanUI#2](https://github.com/SpartanLaboratories/SpartanUI/issues/2): a project site
deployed to GitHub Pages, and the organization's branding. The organization's standard is
[`documentation/dokka.md`](https://github.com/SpartanLaboratories/.github/blob/main/documentation/dokka.md) in
`SpartanLaboratories/.github`.

The rest of the design (the landing page, the branding files, the repository's own documents) rests on facts
about this repository and about the organization's earlier adoptions of the standard, which the plan states with
their sources. One question needs evidence from outside the organization.

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

## Verification results

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
