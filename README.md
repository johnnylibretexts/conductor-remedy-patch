# conductor-remedy-patch

The **Conductor side** of the LibreTexts Remedy accessibility matrix, shipped as
fourteen `git am`-able patches against upstream Conductor.

This repository contains no Conductor source of its own. It is a delivery
vehicle: you clone the real Conductor, apply these commits, and you have the
Project Accessibility WCAG matrix, the remediation work queue and the WCAG
verification panel wired to a Remedy server.

---

## Apply it

```bash
git clone https://github.com/LibreTexts/conductor.git
cd conductor
git checkout -b feat/remedy-a11y-wiring
git am -3 /path/to/conductor-remedy-patch/patches/*.patch
```

That is the whole procedure. The patches were generated against — and verified
to apply cleanly onto — upstream `LibreTexts/conductor` at:

| | |
|---|---|
| Base commit | `1c1a5ec1` |
| Release | `2.144.0` |
| Branch | `master` |

Also verified against upstream `master` at `b0401094` (release `2.151.0`,
2026-09-02): with `-3`, thirteen patches apply untouched and patch 0001 stops on
one trivial conflict in `server/package.json` — upstream changed the adjacent
`"dev"` script line. Keep upstream's `"dev"` line, keep the added `"test"`
line, `git add`, `git am --continue`, and the remaining thirteen go through.

If upstream has moved further and a hunk no longer applies, `git am` stops and
tells you which file. Resolve, `git add`, `git am --continue`. The files most
likely to drift: `server/api.js` (route table), `server/package.json`
(scripts) and `client/src/components/projects/ProjectAccessibility.jsx`.

---

## The twelve commits

Three layers. Each layer is usable without the ones after it.

**Matrix (0001–0003)** — the original delivery.

| # | Commit | What it does |
|---|---|---|
| 1 | `feat(a11y): carry CXone pageID on review sections, extract TOC build/merge` | Adds `pageID` to review sections so a section can be traced back to a real CXone page, and pulls TOC build/merge out into reusable functions. |
| 2 | `feat(a11y): add Remedy scan/preview/apply endpoints and bulk section updates` | The server half. Four new routes plus bulk section item updates. |
| 3 | `feat(a11y): add WCAG matrix, scoring, and Remedy review UI to Project Accessibility` | The client half. The matrix itself, the scoring module, and the preview/apply workflow. |

**Work queue (0004–0007)** — scan findings become persistent work items.

| # | Commit | What it does |
|---|---|---|
| 4 | `feat: connect remediation tasks to reviewed fixes and recovery` | `RemediationWorkItem` model, `GET/POST/PATCH …/accessibility/workqueue`, per-page actions (`scan`, `rendered-scan`, `apply`, `restore`) proxied to the bridge, and the queue + page-action UI under the matrix. |
| 5 | `docs: document remediation release integration and validation limits` | `docs/remediation-integration.md`. |
| 6 | `Refresh remediation matrix and support reviewed complex-image descriptions` | Matrix refreshes only after the scan persisted; evidence attaches to the selected task; reviewed long descriptions for complex images. |
| 7 | `Document book remediation acceptance and static scanner limitations` | Docs: what a static scan cannot see. |

**WCAG verification (0008–0010)** — human evidence, separate from the automated score.

| # | Commit | What it does |
|---|---|---|
| 8 | `feat: require reviewer evidence for WCAG conformance` | `ConformanceRecord` (append-only), the 50-criterion checklist (`conformanceCriteria.json`), `GET …/workqueue/conformance`, `POST …/conformance/{scope,review,signoff}`, and the `ConformanceReview` panel. Reviewer identity and time are set server-side. |
| 9 | `fix: invalidate failed scans and serve authenticated report downloads` | A failed scan cannot count as evidence; `GET …/conformance/report` is authenticated. |
| 10 | `feat: separate remediation handoff acceptance from WCAG verification` | "Delivery accepted" is a distinct state from "criterion verified"; one cannot imply the other. |

**Deployment fixes (0011–0012)**

| # | Commit | What it does |
|---|---|---|
| 11 | `fix: honor configured Conductor browser origins` | CORS seeds its allow-list from `PRODUCTIONURLS` even when `NODE_ENV` is unset. Without this, browser-driven scans from a deployed Commons origin are rejected. |
| 12 | `test(a11y): run RemediationPageActions under vitest` | The React test used `node:test`, which vitest cannot bundle; `npm test` in `client/` now passes. |

**Apply-path safeguards (0013–0014)**

| # | Commit | What it does |
|---|---|---|
| 13 | `fix(a11y): require a reviewed preview before apply and report empty writes` | The matrix Fix modal now requires the same "I reviewed this preview" acknowledgement the work queue already had, and the work queue reports "No changes were written" when the bridge's `applied` flag is false instead of claiming success. |
| 14 | `feat(a11y): name the Conductor reviewer in bridge write requests` | Page actions forward `requested_by` (the authenticated Conductor uuid) so the bridge audit log names the reviewer, not just the server account. Needs the matching bridge change (`requested_by` support); harmless without it. |

Applying only commits 1 and 2 gives you a working API with no UI, which is a
reasonable way to test the server integration on its own. Stopping after 3
gives you the matrix as originally shipped; after 7, the work queue; after 10,
verification; after 14, the apply-path safeguards.

### Files touched

```
client/src/components/projects/ProjectAccessibility.jsx    +1417   the matrix UI
client/src/components/projects/RemediationWorkQueue.tsx     +87   work queue panel (new)
client/src/components/projects/RemediationPageActions.tsx  +114   per-page scan/apply/restore (new)
client/src/components/projects/ConformanceReview.tsx        +90   WCAG verification panel (new)
client/src/components/projects/accessibilityScore.ts       +155   scoring (new)
client/src/components/projects/Projects.css                +382   matrix styles
client/src/types/a11y.ts                                    +82   review shape (new)
client/src/remediation-queue-entry.tsx                      +26   queue mount point (new)
client/src/components/projects/*.test.ts                   +215   tests
server/api/projects.js                                     +426   scan/preview/apply
server/api/remediationqueue.ts                              +83   work-item routes (new)
server/api/remediationactions.ts                            +76   page actions → bridge (new)
server/api/conformance.ts                                   +53   verification routes (new)
server/util/a11yreviewutils.ts                             +105   schema + TOC merge
server/util/remediationqueue.ts                             +42   RemediationWorkItem model (new)
server/util/conformance.ts                                  +60   ConformanceRecord model (new)
server/util/conformanceCriteria.json                       +452   WCAG 2.1 A/AA checklist (new)
server/util/remediationHandoff.ts                           +40   handoff acceptance rules (new)
server/util/*.test.ts                                      +424   tests
server/api.js                                               +55   route registration + CORS
server/models/project.ts                                     +5   stored remedy fields
server/package.json                                          +1   server `npm test`
docs/remediation-integration.md                             +77   (new)
docs/accessibility-verification.md                          +19   (new)
```

Roughly 4,400 lines across 31 files. No dependencies are added — the Remedy
client is plain `fetch` against the server's HTTP API.

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
| GET / POST | `/project/:projectID/accessibility/workqueue` | List / create work items (page, criterion, finding evidence, owner, status). |
| PATCH | `/project/:projectID/accessibility/workqueue/:itemID` | Change status, owner, note; history is appended, not rewritten. |
| POST | `/project/:projectID/accessibility/workqueue/page/:sectionID/:action` | `scan`, `rendered-scan`, `apply`, `restore` — proxied to the Remedy bridge; evidence attaches to an `itemID` if given. |
| GET | `/project/:projectID/accessibility/workqueue/conformance` | The scoped WCAG verification report. |
| GET | `/project/:projectID/accessibility/workqueue/conformance/report` | Authenticated export of that report. |
| POST | `/project/:projectID/accessibility/workqueue/conformance/:action` | `scope`, `review`, `signoff`. Reviewer identity and timestamp are server-set; records are append-only. |

The work-queue and conformance routes reuse the project's edit permission
check. Nothing in them writes to CXone directly — every write goes through the
bridge, which enforces its own sandbox guard.

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
cd server && npm test      # tsx --test — 24 tests: a11yreviewutils, remediation queue/actions, conformance, handoff, CORS
cd client && npm test      # vitest — 19 tests: accessibilityScore, RemediationPageActions
```

Both suites, `tsc --noEmit` and `vite build` in `client/` were run on the
patched tree at base `1c1a5ec1` before this set was published. The line counts
above are net additions measured on that tree.

The scoring tests are worth reading before changing anything: the denominator
counts only pass/fail criteria. Manual-review and not-tested criteria are
reported separately rather than folded in as failures, so an unreviewed project
scores `null` ("not scored") instead of 0%.

---

## Why patches instead of a branch

The branch these came from lives in a private development fork alongside
unrelated deployment work. Exporting the three commits keeps the delivery to
exactly the accessibility matrix and nothing else.

Patches 0001–0003 are byte-identical to that branch, with one deliberate
change: two `node_modules` symlinks pointing at an absolute local path were
committed by accident and have been stripped. They resolved on one machine and
would have broken every other checkout.

Patches 0004–0010 are cherry-picks of the fork's later remediation commits onto
that base (`-x` trailers name the originals). 0011 drops the fork's
`docker-compose.yml` hunk, which belonged to one deployment, and the test that
read it. 0012 was new to this set.

0013–0014 come from `fix/a11y-apply-guards` (johnnylibretexts/conductor-dev
PR #8). 0013 drops that commit's removal of the two committed `node_modules`
symlinks, because this patch set never carried them.
