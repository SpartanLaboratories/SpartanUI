# Accessibility audit of the project site: automatic or by hand, and how

## Question

Should the accessibility audit of the static project site (`website/index.html`, `website/404.html`, `assets/styles.css`, `assets/site.js`) run automatically in the repository's JavaScript test toolchain, or by hand in a browser; and how exactly should the audit and the keyboard-only pass be carried out so that they cover colour contrast and accessible names in both colour themes, on the page served locally before the merge and again on the live site after the first deployment?

Requirements an answer must meet: colour contrast and accessible names are covered in both themes (the theme set by the toggle, or by an emulated `prefers-color-scheme`); the skip link and the toggle are covered by keyboard with a visible focus ring; the audit is repeatable on the live site; no browser download is added to CI unless what it gains justifies it.

## Bottom line

- **Run the audit by hand in a real browser, with a pinned axe-core (4.14.1) injected into the page, once per theme; do not add it to CI.** `color-contrast` cannot be evaluated in jsdom (axe-core's README says so, and a trial on this page confirmed it: no violations, no passes, one errored node). This page also defeats axe's contrast check in a real browser: the header and the page background are gradients, and the hero has a decorative overlay, so seven nodes come back as `incomplete`, not `violations`. A plain "no violations" would pass a page that has a real contrast failure.
- **A plain `axe.run()` is therefore not enough; run it twice per theme.** First as it is. Then again after injecting a style that replaces each gradient with its worst-case solid stop colour and hides the hero overlay. In the trial that second run resolved every contrast node except one gradient button, and it surfaced a real failure (the `.eyebrow` text in the light theme, 3.68:1 against 4.5:1) that the plain run had left as `incomplete`.
- **The keyboard-only pass cannot be automated and is where the site fails today.** In the light theme the focus ring (`--brand-soft`, `#103050`) sits on the always-dark header, about 1.3:1 against the header's middle stop; the skip link's ring is the same navy on a navy button. No axe rule detects this.
- **The audit as run on the current files also reports one real violation in both themes:** `link-in-text-block` on the footer link (colour difference 1.13:1 dark, 2.02:1 light, no underline). The design has to fix these three findings (light-theme eyebrow, light-theme focus ring, in-text links) before the audit can pass.
- **Real-browser automation in CI is not justified now.** Playwright with `@axe-core/playwright` would need a static server, a browser (the hosted Ubuntu 24.04 image already carries Chrome, so `channel: "chrome"` avoids a download) and the same gradient workaround, and it still could not judge focus-ring visibility. An optional, cheap addition is an axe-core run in the existing jsdom component tests limited to the name rules (`button-name`, `link-name`, `image-alt`, `html-has-lang`, `document-title`), which catches a deleted `aria-label` with no browser.
- Lighthouse is not the tool: version 13.6.0 runs axe-core 4.14.0 with only `wcag2a` and `wcag2aa`, so it omits the WCAG 2.1 and 2.2 AA tags, and it has the same gradient limitation.

## Findings

### 1. axe-core does not evaluate colour contrast in jsdom

- Tier: established (the library's own documentation and maintainers), confirmed by a trial.
- axe-core 4.14.1 README, section on supported browsers: "Currently the `color-contrast` rule is known not to work with JSDOM." Source: [axe-core README at v4.14.1](https://github.com/dequelabs/axe-core/blob/v4.14.1/README.md). The maintainers' list of rules that give no useful result in jsdom (issue [#4021](https://github.com/dequelabs/axe-core/issues/4021)) also names `color-contrast-enhanced`, `link-in-text-block`, `target-size`, `scrollable-region-focusable`, `focus-order-semantics`, `frame-focusable-content`, `css-orientation-lock` and `avoid-inline-spacing`; the thread adds that jsdom does not load external CSS by default and has no media-query support.
- jsdom's own README lists "Layout: the ability to calculate where elements will be visually laid out as a result of CSS" as out of scope and says jsdom "still does not do any layout or rendering" ([jsdom README, main branch, read 2026-11-23](https://github.com/jsdom/jsdom/blob/main/README.md)).
- Trial (jsdom 31.0.1, axe-core 4.14.1, `index.html` plus `styles.css` inlined, `pretendToBeVisual: true`): `violations: []` in both themes; `color-contrast` appears in neither `passes` nor `violations`; its only entry is one `incomplete` node, `.skip-link`, with the message "Axe encountered an error; test the page for this type of problem manually". A jsdom audit therefore reads as clean while checking nothing about colour.
- Name rules do work in jsdom. With `runOnly: { type: "rule", values: ["button-name", "link-name", "image-alt", "html-has-lang", "document-title", "bypass"] }` the page passes all six; after deleting the toggle's `aria-label` and `title` the same run reports a `button-name` violation.

### 2. In a real browser, axe-core cannot resolve contrast over gradients and overlays, and reports it as `incomplete`

- Tier: established behaviour (axe documents `incomplete` as "needs review"), observed on this page.
- axe-core API: `incomplete` is "Also known as "needs review," these results were aborted and require further testing." ([API.md at v4.14.1](https://github.com/dequelabs/axe-core/blob/v4.14.1/doc/API.md)).
- Trial (axe-core 4.14.1 via `@axe-core/playwright` 4.14.0, `playwright-core` 1.65.1, Microsoft Edge 157 headless, the page served over `http.server`), identical in dark, light set by attribute, light set by the toggle, and light by `colorScheme: "light"` emulation: seven `color-contrast` nodes are `incomplete`. `.brand > span` and `.nav-gh > span` give "could not be determined due to a background gradient" (the header's `--bg-header`); `.eyebrow`, `h1`, `.lede`, `.lede > a` and `.btn` give "due to a pseudo element" (the hero's `::before` glow). Only the footer text was resolved (7.07 and 5.18:1 for the footer paragraph in dark and light).
- Re-run after injecting `html body{background:<solid stop>!important} .site-header{background:#103050!important} .hero::before{display:none!important}` (the stops are the page's own gradient end colours in `styles.css`): contrast resolved for every node except `.btn`, and the light theme then reports a real violation: `.eyebrow`, "insufficient color contrast of 3.68 (foreground color: #907040, background color: #ece6d6 ...). Expected contrast ratio of 4.5:1". Dark-theme ratios against the lightest stop `#16263a` were all above 6.2:1.
- `.btn` stays `incomplete` because its own background is a gradient; white text on its lightest stop `#1a4673` is about 9.7:1 (computed from the WCAG relative-luminance formula), so it can be closed by hand.
- Consequence: for this page "no `violations`" is a necessary, not a sufficient, pass criterion. Every `color-contrast` `incomplete` node must be resolved, either by the flattening run or by hand.

### 3. The names and the skip link are checkable by axe in a real browser

- Tier: common practice; observed.
- `button-name` (tags `wcag2a`, `wcag412`), `link-name` (`wcag2a`, `wcag244`, `wcag412`), `image-alt` and `svg-img-alt` (`wcag2a`, `wcag111`), `bypass` (`wcag2a`, `wcag241`) exist in 4.14.1 (checked with `axe.getRules()`). The `skip-link` rule carries only `best-practice`, so a WCAG-tags-only run omits it; add the `best-practice` tag to include it and `landmark-one-main`.
- Trial: with `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa` and `best-practice`, `bypass`, `skip-link`, `button-name`, `link-name`, `image-alt`, `document-title`, `html-has-lang` and `landmark-one-main` all pass in both themes; `svg-img-alt` is not applicable because the SVG icons are `aria-hidden`.

### 4. `runOnly` tags: one tag per rule for the WCAG version and level, so list them all

- Tier: established (documentation).
- API.md at v4.14.1: "Each rule has one tag that indicates which WCAG version / level it belongs to"; "To run both WCAG 2.0 Level A and Level AA rules, you must specify both `wcag2a` and `wcag2aa`". The tag table lists `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa` (there is no `wcag22a`). The set of those five tags selects 70 rules in 4.14.1 (counted with `axe.getRules()`); `best-practice` adds 30 more.
- The result object has "a `passes` array, a `violations` array, an `incomplete` array and an `inapplicable` array."

### 5. Current release and CDN

- Tier: established (registry and CDN responses, 2026-11-23).
- `npm view axe-core` returns `latest: 4.14.1` (modified 2026-11-16); `@axe-core/playwright` is 4.14.0, with a `playwright-core >= 1.0.0` peer dependency.
- `https://cdn.jsdelivr.net/npm/axe-core@4.14.1/axe.min.js` returns 200 with `x-jsd-version: 4.14.1`, `access-control-allow-origin: *` and `cache-control: public, max-age=31536000 ... immutable`; the file begins `/*! axe v4.14.1`. Its SHA-384 (computed by hand on 2026-11-23) is `xt1SFg49QaDwsyt0Xfs2XAyBtYPClXl3X2q5OOLeGAUmq9jZTYIMdztpF99WdhiH`, usable as a Subresource Integrity value. Injecting the script into the page from the browser's JavaScript with that `integrity` and `crossOrigin = "anonymous"` worked on `http://localhost`.

### 6. Lighthouse runs axe-core, but a narrower rule set

- Tier: common practice (read from the tool's own source).
- Lighthouse 13.6.0 depends on `"axe-core": "4.14.0"` ([package.json at v13.6.0](https://github.com/GoogleChrome/lighthouse/blob/v13.6.0/package.json)) and its accessibility gatherer calls `axe.run` with `runOnly: { type: 'tag', values: ['wcag2a', 'wcag2aa'] }` ([core/gather/gatherers/accessibility.js at v13.6.0](https://github.com/GoogleChrome/lighthouse/blob/v13.6.0/core/gather/gatherers/accessibility.js)). It therefore skips `wcag21aa`, `wcag22aa` and `best-practice` (so no `skip-link` rule). Its score "is a weighted average" with weighting based on axe user impact ([Chrome docs, Lighthouse accessibility score](https://developer.chrome.com/docs/lighthouse/accessibility/scoring)). The CLI needs Node 22 or later and a local Chrome. It offers nothing the direct axe-core run does not, and its pass/fail audits show a gradient-covered node as "needs review" the same way.

### 7. What automation misses, so the keyboard pass stays manual

- Tier: common practice; one national-government study plus the page's own trial.
- The UK Government Digital Service tested 10 automated tools on a page with known barriers: "a large proportion of the barriers we created weren't picked up by any of the 10 tools we tested", 29% ([GDS blog, 2017-02-24](https://accessibility.blog.gov.uk/2017/02/24/what-we-found-when-we-tested-tools-on-the-worlds-least-accessible-webpage/)). It advises combining automated tools with manual checking. The figure is from 2017 and tool-dependent; read it as direction, not as a rate for axe 4.14.
- On this page the trial confirms the pattern: axe reports the skip link, the toggle's name and the landmarks as passing, while the light-theme focus ring is effectively invisible. Screenshots of the focused skip link and toggle in the light theme show no discernible ring on the dark header.

### 8. Cost of a real browser in CI

- Tier: established (vendor documentation).
- Playwright's browsers "will take a few hundred megabytes of disk space when installed" (Chromium about 281 MB in its example), downloaded by `npx playwright install`, with `install --with-deps chromium` for operating-system libraries on CI ([Playwright, Browsers](https://playwright.dev/docs/browsers)). It can drive an installed Chrome or Edge instead: "Playwright can operate against the branded Google Chrome and Microsoft Edge browsers available on the machine."
- The GitHub-hosted `ubuntu-24.04` image (the `ubuntu-latest` label) lists "Google Chrome 157.0.8103.78" and Microsoft Edge 157 ([runner-images Ubuntu2404-Readme](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md)), so a CI audit could avoid the download with `channel: "chrome"`. The trial used `playwright-core` with the machine's Edge and no browser download.

### 9. The success criteria the pass maps to

- Tier: established (W3C).
- 2.4.1 Bypass Blocks (Level A): "A mechanism is available to bypass blocks of content that are repeated on multiple web pages." 2.1.1 Keyboard (A): "All functionality of the content is operable through a keyboard interface". 2.4.7 Focus Visible (AA): "Any keyboard operable user interface has a mode of operation where the keyboard focus indicator is visible." 1.4.3 Contrast (Minimum) (AA): "at least 4.5:1", large-scale text "at least 3:1". 1.4.11 Non-text Contrast (AA): "Visual information required to identify user interface components and states" needs 3:1 against adjacent colours. 4.1.2 Name, Role, Value (A): "the name and role can be programmatically determined". Sources: the [W3C Understanding pages for WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/).
- 2.4.7 requires only that the indicator be visible; it states no ratio. The working threshold used here is 3:1 against the adjacent colours, which is what 1.4.11 asks for components and states.

## How comparable projects do it

- **axe-core's own repository** keeps jsdom support in a separate `test:jsdom` script and its jsdom example disables the rule: `rules: { 'color-contrast': { enabled: false } }` ([doc/examples/jsdom/test/a11y.js at v4.14.1](https://github.com/dequelabs/axe-core/blob/v4.14.1/doc/examples/jsdom/test/a11y.js)). Contrast is expected to be checked in a real browser.
- **Lighthouse** gathers axe-core results inside the page being audited (`axe.run(document, { runOnly: { type: 'tag', values: ['wcag2a', 'wcag2aa'] }, ... })`, `Accessibility` gatherer, v13.6.0), the same in-page injection the recommended manual audit uses.
- **`@axe-core/playwright` 4.14.0** is the standard way to run axe in an automated real browser (`new AxeBuilder({ page }).withTags([...]).analyze()`), with `playwright-core` as a peer; it needs a served page and a browser. It was used in the trial against the system Edge.

## Pitfalls & regrets

- **A green `violations` list that hides `incomplete`.** The most likely mistake on this page: seven contrast nodes are `incomplete` in a plain run and one of them is a genuine failure. Treat any non-empty `color-contrast` `incomplete` as an open item.
- **jsdom looks supported and is not for contrast.** Running the whole rule set there gives a false all-clear; limit a jsdom run to name rules.
- **Theme persistence.** `site.js` stores the choice in `localStorage` (`spartanui-theme`) and reads it before the system preference. After the toggle has been pressed once, emulating `prefers-color-scheme` changes nothing. Clear the key (`localStorage.removeItem("spartanui-theme")`) and reload before testing the system-preference path, and keep the toggle path and the emulation path in separate fresh page loads.
- **The flattening run's stop colours are copied from `styles.css`.** If the gradients change, the injected values must change with them; a stale value can hide a failure.
- **Gradient-clipped text on the 404 page.** `.grad` is `color: transparent` over `background-clip: text`; axe cannot judge it and large text only needs 3:1, so check both gradient stops against the page background by hand.
- **Fragment targets that are not focusable.** `<main id="main">` has no `tabindex`. In Chromium (Edge 157, trial) following the skip link leaves `document.activeElement` on `body` but moves the next Tab to the first focusable element after the target, which works. Other browsers have differed historically; an open point is whether to add `tabindex="-1"` to `main`. This was not tested outside Chromium.
- **The live 404 page.** It renders styled only under `/SpartanUI/`, so it is audited on the live site only (any unknown path such as `/SpartanUI/does-not-exist` serves it). It loads no `site.js`, so it is always dark.

## Recommendation for this repository

**Mode: by hand in a real browser, axe-core 4.14.1 injected from the CDN, once per theme, plus a keyboard-only pass; nothing added to CI for contrast.** The page's contrast cannot be settled by any tool that is cheap to run in CI: jsdom cannot compute it at all, and a real browser leaves most of it `incomplete` until a worst-case flattening step is applied, which belongs with a person or a browser pane that reads the result. The focus-ring visibility, the skip link's landing point and the Space/Enter behaviour of the toggle need eyes or screenshots anyway. A CI browser job (about 280 MB download, or the preinstalled Chrome plus a server step) would only add regression protection for the name rules, which jsdom already covers for free.

Optional, at no browser cost: one component test that runs `axe.run` in the existing jsdom environment with `runOnly` set to the name rules (`button-name`, `link-name`, `image-alt`, `html-has-lang`, `document-title`) and asserts no violations. This depends on the separate choice of the test runner and DOM emulator; it adds the `axe-core` package as a dev dependency. If the site grows past a page or two, revisit a Playwright job using the runner's Chrome.

### Serving the page

Local: `python3 -m http.server 8000 --directory _site`, page at `http://localhost:8000/`. Live: `https://spartanlaboratories.github.io/SpartanUI/`. The steps below are identical on both; on the live site also run the audit once on the 404 page (dark only, step 5).

### Keyboard-only pass (a person, or a browser pane that can send key presses)

Do this on the home page once in each theme. Start each theme from a fresh load; for the light theme either press the toggle (step 4) or, with `localStorage.removeItem("spartanui-theme")` run and the page reloaded, emulate `prefers-color-scheme: light` (DevTools, Rendering panel). Do not use the mouse.

1. Load the page. Press Tab once. Expect: the "Skip to content" link appears at the top-left and has a ring that can be seen against what is behind it. Read `getComputedStyle(document.activeElement)`: `outlineStyle` is `solid`, `outlineWidth` `2px`.
2. Press Enter. Expect: the address ends `#main`. Press Tab once more. Expect: focus lands on the first link inside the main content ("library split plan"), not on the brand, GitHub or the toggle. That is the bypass working (2.4.1).
3. Reload. Press Tab four times. Expect, in order: skip link, brand ("SpartanUI"), GitHub, theme toggle. Focus is on `button.theme-toggle` with a ring clearly visible against the header (the header is dark ink in both themes). This is the check that fails in the light theme today: the ring colour `rgb(16, 48, 80)` is nearly the colour of the header behind it. Pass needs about 3:1 against the header (2.4.7, 1.4.11).
4. With the toggle focused, press Space. Expect `document.documentElement.dataset.theme` to flip. Press Enter. Expect it to flip back. Both keys work (2.1.1, 4.1.2). Leave the theme you want to test next.
5. Keep Tabbing to the end: remaining stops are the "library split plan" link, "View on GitHub" and the footer licence link, each with a visible ring on its own background (the hero and footer use the page background, so the light-theme navy ring is fine there). Shift+Tab walks back in the reverse order.
6. Record, per theme, one screenshot of the focused skip link and one of the focused toggle.

Pass: every step shows the expected result in both themes, and every ring is visible by eye at 100% zoom. Any ring that cannot be seen on its background is a failure.

### Audit steps (a person in the browser console, or a pane that executes JavaScript in the page)

Do each theme in a fresh page load. Tool: axe-core 4.14.1.

**A. Load axe (same for every run).** Run in the page's console. The SRI value is the file's SHA-384 for 4.14.1; the script fails to load if the file differs.

```js
await new Promise((ok, no) => {
  const s = document.createElement('script');
  s.src = 'https://cdn.jsdelivr.net/npm/axe-core@4.14.1/axe.min.js';
  s.integrity = 'sha384-xt1SFg49QaDwsyt0Xfs2XAyBtYPClXl3X2q5OOLeGAUmq9jZTYIMdztpF99WdhiH';
  s.crossOrigin = 'anonymous'; s.onload = ok; s.onerror = no; document.head.append(s);
});
axe.version  // must read "4.14.1"
```

**B. Pick the theme.** Dark: the default on load with a dark system preference and no stored choice. Light: press the toggle (click, or Tab to it and press Enter), or set `localStorage.removeItem("spartanui-theme")`, reload and emulate `prefers-color-scheme: light`. Confirm with `document.documentElement.dataset.theme`. Load axe (A) after the theme is set, or again after a reload.

**C. Plain run.**

```js
window.__r = await axe.run(document, { runOnly: { type: 'tag', values:
  ['wcag2a','wcag2aa','wcag21a','wcag21aa','wcag22aa','best-practice'] } });
const f = a => a.map(v => ({ id: v.id, nodes: v.nodes.map(n => n.target.join(' ') + ' :: ' + (n.any[0]?.message || n.none[0]?.message || n.failureSummary)) }));
JSON.stringify({ theme: document.documentElement.dataset.theme, violations: f(__r.violations), incomplete: f(__r.incomplete) }, null, 1)
```

Plain-run pass criteria: `violations` is empty; no `violations` entry is `color-contrast`, `button-name`, `link-name`, `image-alt` or `link-in-text-block`; `bypass`, `skip-link`, `button-name`, `link-name` and `image-alt` are in `__r.passes`. The `incomplete` list is expected to hold the gradient and overlay nodes, handled in D; any `incomplete` entry that is not a `color-contrast` node on `.brand > span`, `.nav-gh > span`, `.eyebrow`, `h1`, `.lede`, `.lede > a`, `.btn` or a `link-in-text-block` node on `.lede > a` is new and has to be read and resolved by hand.

**D. Worst-case flattening run (resolves the contrast `incomplete` nodes).** In the same page, add a style that replaces each gradient with its worst-case solid stop, then run axe again for contrast and links only. Dark theme (light text, so the lightest stop is the worst case):

```js
const st = document.createElement('style');
st.textContent = 'html body{background:#16263a!important} .site-header{background:#103050!important} .hero::before{display:none!important}';
document.head.append(st);
const r2 = await axe.run(document, { runOnly: { type: 'rule', values: ['color-contrast','link-in-text-block'] } });
JSON.stringify({ violations: r2.violations.map(v => [v.id, v.nodes.map(n => n.target.join(' '))]), incomplete: r2.incomplete.map(v => [v.id, v.nodes.map(n => n.target.join(' '))]) })
```

For the light theme (dark text, so the darkest stop of the page gradient is the worst case) use `html body{background:#ece6d6!important}` and keep the header and overlay rules (the header is ink in both themes). A second run against the lightest page stop (`#f7f5ef` light, `#101a26` dark) is optional.

D pass criteria: `violations` has no `color-contrast` entry; `incomplete` contains no node other than `.btn` (its own gradient background). Close `.btn` by hand: white text on the lightest stop of its gradient, `#1a4673`, is about 9.7:1, above 4.5:1. Any other `incomplete` node is an open item, not a pass. The overlay is hidden in D; the glow is at most 18% of a brand colour over the page colour and does not alter any ratio by more than a fraction, so require a margin of about 0.5:1 above the threshold for the nodes it covers (`.eyebrow`, `h1`, `.lede`, `.lede > a`) rather than a bare 4.5:1.

**E. Names, both themes.** Already part of C: `button-name` (toggle: `aria-label="Switch colour theme"`), `link-name` (brand, GitHub, hero and footer links), `image-alt` (the logo with `alt=""` is presentational, a pass) must not be in `violations` or `incomplete`. A cross-check by eye: the toggle's accessible name read from the page is "Switch colour theme", and the decorative logo contributes nothing to the brand link's name, which stays "SpartanUI".

**F. The 404 page, on the live site only.** Open `https://spartanlaboratories.github.io/SpartanUI/does-not-exist`. Run A and C (no theme choice exists; it is dark). The `.code` heading is gradient-clipped transparent text, so its contrast shows as `incomplete`; check by hand that both stops of its gradient (`#C0A080` to `#C0A080` in dark) against `#101a26` reach at least 3:1 (large-scale text).

**What to do with `incomplete`.** A result is only a pass when `incomplete` is empty or each remaining node is closed by a stated hand calculation (foreground against the worst-case background, same formula as WCAG 1.4.3). Record node, ratio and the colours used. If a CDN script is blocked on the live site (a content-security policy), fall back to the browser's axe extension at the same engine version, with the same plain-run and flattening steps, or to a local Playwright run (below); this has not been checked for the deployed site.

**Optional scripted equivalent (a Node script, not a CI job):** `playwright-core` plus `@axe-core/playwright`, with `chromium.launch({ channel: 'chrome' })` (or `'msedge'`) so no browser is downloaded, one context per theme (`colorScheme: 'dark'` / `'light'`, or the toggle pressed), `page.addStyleTag` for D, and `new AxeBuilder({ page }).withTags([...]).analyze()`. The trial ran this way against the local page.

### Findings the design has to resolve before the pass can succeed

Observed on the current files with the steps above. These are properties of the page, not of the audit method.

- Light theme, `.eyebrow`: `#907040` on the page background is 3.68:1 on the darkest page stop (`#ece6d6`), below 4.5:1 for 12.5px text. Needs a darker accent for small text in the light theme.
- Both themes, `.footer-inner > p > a` (and the hero link in `.lede`): `link-in-text-block` violation, link colour too close to the surrounding text (1.13:1 dark, 2.02:1 light) and no underline. An underline on links inside running text resolves it.
- Light theme, `:focus-visible` on header elements (skip link, toggle, brand, GitHub): `--brand-soft` is navy `#103050` on a dark-ink header. The focus ring colour for the header needs its own token, as the header already has fixed tokens.
- Open point: the toggle's own border (`--ink-border`, `#2b3f57` on ink) is about 1.5:1. The icon (`--ink-muted`, about 7:1 on the header) identifies the control; whether the border must also reach 3:1 under 1.4.11 is a judgement, not a tool result. Recommendation: raise the border or make the icon the sole identifier explicitly, and note which.

## Load-bearing claims

1. axe-core's colour contrast rule does not work in jsdom. Quote: "Currently the `color-contrast` rule is known not to work with JSDOM." Source: https://github.com/dequelabs/axe-core/blob/v4.14.1/README.md (axe-core 4.14.1; confirmed by the trial).
2. jsdom does no layout or rendering. Quote: "still does not do any layout or rendering". Source: https://github.com/jsdom/jsdom/blob/main/README.md (main branch, read 2026-11-23; trial used jsdom 31.0.1).
3. A WCAG 2.0 A and AA run needs both tags; each rule carries one WCAG tag; `wcag2a`, `wcag2aa`, `wcag21aa` and `wcag22aa` are valid tags. Quote: "To run both WCAG 2.0 Level A and Level AA rules, you must specify both `wcag2a` and `wcag2aa`". Source: https://github.com/dequelabs/axe-core/blob/v4.14.1/doc/API.md (4.14.1). The existence of `wcag21a` and its absence of a `wcag22a` tag were read from the same file's tag table and from `axe.getRules()`.
4. `incomplete` means a result needing further testing, not a pass. Quote: "these results were aborted and require further testing." Source: https://github.com/dequelabs/axe-core/blob/v4.14.1/doc/API.md (4.14.1).
5. axe-core 4.14.1 is the current release and the jsDelivr path serves it. Quote (file banner and response header): "/*! axe v4.14.1" and "x-jsd-version: 4.14.1". Source: https://cdn.jsdelivr.net/npm/axe-core@4.14.1/axe.min.js (fetched 2026-11-23); `npm view axe-core` `dist-tags.latest = 4.14.1`, modified 2026-11-16.
6. Lighthouse 13.6.0 runs axe-core 4.14.0 with only the `wcag2a` and `wcag2aa` tags. Quote: `"axe-core": "4.14.0",` and `values: [ 'wcag2a', 'wcag2aa', ]` (as laid out in the source). Sources: https://github.com/GoogleChrome/lighthouse/blob/v13.6.0/package.json and https://github.com/GoogleChrome/lighthouse/blob/v13.6.0/core/gather/gatherers/accessibility.js (v13.6.0). The Lighthouse inside a given Chrome's DevTools may be a different version; unconfirmed.
7. Automated tools miss a large share of barriers, so manual testing is still needed. Quote: "a large proportion of the barriers we created weren't picked up by any of the 10 tools we tested". Source: https://accessibility.blog.gov.uk/2017/02/24/what-we-found-when-we-tested-tools-on-the-worlds-least-accessible-webpage/ (2017-02-24).
8. Playwright's browser download is a few hundred megabytes and an installed Chrome or Edge can be used instead. Quotes: "will take a few hundred megabytes of disk space when installed" and "Playwright can operate against the branded Google Chrome and Microsoft Edge browsers available on the machine". Source: https://playwright.dev/docs/browsers (Playwright 1.65.1 era, read 2026-11-23).
9. The `ubuntu-latest` hosted runner image carries Google Chrome. Quote: "Google Chrome 157.0.8103.78". Source: https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md (main, read 2026-11-23); the mapping `ubuntu-latest` to Ubuntu 24.04 is from https://github.com/actions/runner-images/blob/main/README.md. Image contents change between releases.
10. WCAG 2.2 success criteria. Quotes: "A mechanism is available to bypass blocks of content that are repeated on multiple web pages." (2.4.1, A); "Any keyboard operable user interface has a mode of operation where the keyboard focus indicator is visible." (2.4.7, AA); "have a contrast ratio of at least 3:1 against adjacent color(s)" (1.4.11, AA); "at least 4.5:1" and "at least 3:1" for large-scale text (1.4.3, AA); "the name and role can be programmatically determined" (4.1.2, A). Sources: https://www.w3.org/WAI/WCAG22/Understanding/bypass-blocks.html, .../focus-visible.html, .../non-text-contrast.html, .../contrast-minimum.html, .../name-role-value.html, .../keyboard.html (WCAG 2.2).
11. Lighthouse's accessibility weighting is based on axe impact, which supports "Lighthouse is axe-based". Quote: "Weighting is based on" (followed by the link text "axe user impact assessments"). Source: https://developer.chrome.com/docs/lighthouse/accessibility/scoring (read 2026-11-23). The page does not say "axe-core" in words; the engine claim rests on claim 6.
12. GitHub Pages sends no content-security policy that blocks the CDN script on project sites: unconfirmed. No deployed site existed to test; `https://pages.github.com/` returned no such header, which is not evidence for a project site.

## Gaps

- The site is not deployed (`https://spartanlaboratories.github.io/SpartanUI/` returns 404 on 2026-11-23), so the live-site steps, the live 404 page and the CDN-injection path under real Pages headers were not run. The local trial used a plain `http.server`.
- The trial used Microsoft Edge 157 (Chromium) with Playwright 1.65.1, not Chrome or another browser. Firefox and Safari focus behaviour after a skip link (the `main` has no `tabindex`) was not tested.
- The axe DevTools browser extension and the Lighthouse panel in DevTools were not run, so their rule sets and versions are unconfirmed; only the Lighthouse source at 13.6.0 was read.
- The 404 page was not audited by a tool (it renders unstyled locally because its URLs are absolute under `/SpartanUI/`).
- The flattening stops and ratios (`#16263a`, `#ece6d6`, `#103050`, 9.7:1, about 1.3:1 and about 7:1) were computed from `styles.css` as of the trial; the focus-ring ratio is a hand calculation against the header's middle stop, not a measured pixel value, and the toggle border question under 1.4.11 is a judgement.
- The 3:1 threshold for focus rings is an interpretation (2.4.7 states no ratio); the W3C text read does not itself name focus indicators under 1.4.11.
- Screen-reader behaviour (announcement of the toggle's state, which flips only visually) was not tested. The toggle has no `aria-pressed` or state in its name; whether that is acceptable is a design question not settled here.
