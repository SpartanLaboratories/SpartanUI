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
