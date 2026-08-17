# conductor-remedy-patch

The **Conductor side** of the LibreTexts Remedy accessibility matrix, shipped as
three `git am`-able patches against upstream Conductor.

This repository contains no Conductor source of its own. It is a delivery
vehicle: you clone the real Conductor, apply these three commits, and you have
the Project Accessibility WCAG matrix wired to a Remedy server.

---

## Apply it

```bash
git clone https://github.com/LibreTexts/conductor.git
cd conductor
git checkout -b feat/remedy-a11y-wiring
git am /path/to/conductor-remedy-patch/patches/*.patch
```

That is the whole procedure. The patches were generated against — and verified
to apply cleanly onto — upstream `LibreTexts/conductor` at:

| | |
|---|---|
| Base commit | `1c1a5ec1` |
| Release | `2.144.0` |
| Branch | `master` |

If upstream has moved on and a hunk no longer applies, `git am` stops and tells
you which file. Resolve, `git add`, `git am --continue`. Only two files are
likely to drift: `server/api.js` (route table) and
`client/src/components/projects/ProjectAccessibility.jsx`.

---

## The three commits

| # | Commit | What it does |
|---|---|---|
| 1 | `feat(a11y): carry CXone pageID on review sections, extract TOC build/merge` | Adds `pageID` to review sections so a section can be traced back to a real CXone page, and pulls TOC build/merge out into reusable functions. |
| 2 | `feat(a11y): add Remedy scan/preview/apply endpoints and bulk section updates` | The server half. Four new routes plus bulk section item updates. |
| 3 | `feat(a11y): add WCAG matrix, scoring, and Remedy review UI to Project Accessibility` | The client half. The matrix itself, the scoring module, and the preview/apply workflow. |

Applying only commits 1 and 2 gives you a working API with no UI, which is a
reasonable way to test the server integration on its own.

### Files touched

```
client/src/components/projects/ProjectAccessibility.jsx   +1429   the matrix UI
client/src/components/projects/accessibilityScore.ts       +155   scoring (new)
client/src/components/projects/accessibilityScore.test.ts  +162   tests (new)
client/src/components/projects/Projects.css                +383   matrix styles
client/src/types/a11y.ts                                    +82   review shape (new)
server/api/projects.js                                     +460   scan/preview/apply
server/util/a11yreviewutils.ts                             +106   schema + TOC merge
server/util/a11yreviewutils.test.ts                        +130   tests (new)
server/api.js                                               +48   route registration
server/models/project.ts                                     +5   stored remedy fields
server/package.json                                          +1   server `npm test`
```

Roughly 2,900 lines. No dependencies are added — the Remedy client is plain
`fetch` against the server's HTTP API.

---

## Integration surface

### Routes added

All under the existing authenticated project API:

| Method | Path | Purpose |
|---|---|---|
| GET | `/project/accessibility/remedy/health` | Is the Remedy server reachable? Drives the health indicator in the UI. |
| POST | `/project/accessibility/remedy/scan` | Scan a section's CXone page, store findings + WCAG summary. |
| POST | `/project/accessibility/remedy/preview` | Ask Remedy for a proposed fix. Returns an HTML diff and a `previewToken`. |
| POST | `/project/accessibility/remedy/apply` | Apply a previously previewed fix. Requires the `previewToken`. |
| PUT | `/project/accessibility/review/section/items` | Bulk-update checkbox matrix items for a section. |

**Apply is only reachable after a preview.** The client sends the preview hash
back and the server rejects a stale one, so a page that changed between preview
and apply fails closed rather than overwriting someone's edit.

### Environment variables

Server:

| Variable | Notes |
|---|---|
| `REMEDY_API_BASE_URL` | Remedy server base URL. `PROJECT_REMEDY_BASE_URL` is accepted as an alias. |
| `REMEDY_API_KEY` | Sent as the `X-API-Key` header. `PROJECT_REMEDY_API_KEY` is accepted as an alias. |

Client (Conductor's `CLIENT__*` passthrough):

| Variable | Notes |
|---|---|
| `CLIENT__REMEDY_ENABLED` | `"true"` to show the Remedy controls in the matrix. |
| `CLIENT__REMEDY_PANEL_URL` | Public URL of the Remedy bridge staff panel. |

If `REMEDY_API_KEY` is unset the header is simply omitted, which is fine against
a Remedy server that does not require one.

### Compose wiring

Conductor has to be able to resolve the Remedy server by name, which means
joining Remedy's external Docker network:

```yaml
services:
  conductor:
    environment:
      REMEDY_API_BASE_URL: http://remedy-server:8000
      REMEDY_API_KEY: ${REMEDY_API_KEY}
      CLIENT__REMEDY_ENABLED: "true"
      CLIENT__REMEDY_PANEL_URL: https://remedy.example.org
    networks: [cond, remedy_net]

networks:
  remedy_net:
    external: true
    name: libretexts_remedy
```

Forgetting the `networks:` entry is the most common failure: the app starts
fine, and the health endpoint just reports the Remedy server as unreachable.

---

## The rest of the stack

Conductor is one client of a four-part system. The other three are separate
repositories:

| Repository | Role |
|---|---|
| `libretexts-remedy-server` | The engine. All checking and remediation behind `/v1/*`. Everything else is a client of it. |
| `libretexts-remedy-bridge` | CXone Expert page I/O — reads page HTML, writes approved revisions through guarded APIs. |
| `adapt-a11y-scanner` | Renders ADAPT questions in headless Chromium (MathJax-aware) and posts the real DOM to the server. |

**Read `deploy/stack/README.md` in `libretexts-remedy-server` before deploying.**
It has the only compose file that runs all three together, and the failure modes
that are not obvious from the individual READMEs. Running the Conductor
accessibility matrix needs all three services.

---

## Tests

```bash
cd server && npm test      # tsx --test, covers a11yreviewutils
cd client && npm test      # vitest, covers accessibilityScore
```

The scoring tests are worth reading before changing anything: the denominator
counts only pass/fail criteria. Manual-review and not-tested criteria are
reported separately rather than folded in as failures, so an unreviewed project
scores `null` ("not scored") instead of 0%.

---

## Why patches instead of a branch

The branch these came from lives in a private development fork alongside
unrelated deployment work. Exporting the three commits keeps the delivery to
exactly the accessibility matrix and nothing else.

The patches are byte-identical to that branch, with one deliberate change: two
`node_modules` symlinks pointing at an absolute local path were committed by
accident and have been stripped. They resolved on one machine and would have
broken every other checkout.
