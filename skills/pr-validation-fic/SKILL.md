---
name: pr-validation-fic
description: "Use when validating a SharePoint client PR by capturing BEFORE/AFTER screenshots — reads PR description + changed files, writes a custom spec per PR (creates pages, seeds REST data, toggles list settings, pulls in a second user as needed), runs it, returns screenshots. Keywords: PR validation, BEFORE/AFTER screenshot, Panel→OverlayDrawer migration, sp-pages-playwright, debug link, debugFlights, PR build CDN, FIC synthetic tenant"
---

# SharePoint PR Visual Validation — BEFORE/AFTER in One Shot

Give a PR number. Get two fullpage screenshots (`BEFORE` = prod CDN render, `AFTER` = PR build render via debug link) of the PR's UI surface on the **FIC synthetic tenant**. ~3 minutes per PR for a simple trigger, longer for multi-user / complex triggers.

**Scope:** client-only SharePoint PRs that change a SitePage component (Panel→Drawer migrations, social-bar buttons, command-bar items, comment-area panels, etc).

**Out of scope:** server-side changes, NGSP chrome (white SuiteNav / canvas shadow / large-icon AppBar) — these require server-side opt-in that the synthetic tenant doesn't have. See "Gotcha #6".

---

## What this skill actually does

This is **not** a "run a fixed pre-existing spec against a fixed page" flow. When invoked, Claude:

- **Reads the PR description + changed files** to infer where the changed UI lives, what triggers it, and what setup it needs (data seeding, list settings, second user, web part dependency, etc.)
- **Picks the right reference pattern** (A / B / C / D in Step 3) from `src/test/PRValidation/` and adapts it — not a single hardcoded flow
- **Writes a brand-new spec per PR** with the right page setup, REST calls, trigger selector, and probe assertions
- **Mutates tenant state when needed** (create page, enable list moderation, add web part, post comment as another user, etc.) — and **self-cleans in a `finally` block** so the synthetic tenant is left clean for the next run
- **Runs the spec, reads probe logs, captures BEFORE/AFTER screenshots**, moves them to the curated folder

Surfaces that need list-setting changes, multi-user setup, or REST-seeded data are **in scope** — the skill is allowed to drive all of that within the spec. Even Pattern D (3rd-party web part dependency) is **attempted first** — only after a quick reachability probe proves the dependency is genuinely not registered on the synthetic tenant does the skill fall back to manual handoff.

---

## Prerequisites (read once before first run)

### What to install
1. **odsp-web repo** — this skill runs inside the repo, not as a standalone tool
2. **Azure CLI** + login: `az login` (Step 1's `az account get-access-token` needs it; pick the Microsoft org/tenant)
3. **Rush ready**: `rush install` has been run at repo root
4. **Network access to** `odspwebcidev.z13.web.core.windows.net` (PR build CDN) — corp VPN usually fine

### Anything to set up on FIC?
**Basically no.** The synthetic tenant is shared and pre-provisioned:

- ✅ Tenant `a830edad90508492pxhesjk9iuz.sharepoint.com` already exists
- ✅ Three test users (adminUser / nonAdminUser / globalReaderUser) already created
- ✅ FIC auth handled by `playwright-utilities` fixtures automatically — no manual login
- ✅ No cookie / token / storageState management needed

**Only case that needs a human:** after the skill has actually probed and confirmed the dependency is not registered on the synthetic tenant (Pattern D, e.g. Planner web part missing from the picker, or a server-side product entry point returning access-denied). In that case, ask the dependency owner to register it on the synthetic tenant, or skip and let the PR author screenshot manually. **Never skip a Pattern D PR without probing first** — many surfaces look external but are actually reachable (e.g. Viva Amplify is reachable via `/_layouts/15/viva-amplify.aspx` on the synthetic tenant).

### First-run troubleshooting
| Symptom | Cause | Fix |
|---|---|---|
| Step 1 curl returns 401 | `az login` didn't pick Microsoft org | `az login --tenant <microsoft-tenant-id>` |
| Spec compile: module not found | `rush install` not run | `rush install` at repo root |
| AFTER `prBuildCount=0` even though ADO works | VPN off, CDN unreachable | connect VPN, re-run |
| `rushx playwright` command not found | Not in sp-pages-playwright dir | `cd sp-client/integration-tests/sp-pages-playwright` |

---

## Quick start (TL;DR)

```
1. Get PR debug link        → 1 curl  (Step 1)
2. Pick flights             → usually just '1535'  (Step 2)
3. Identify trigger pattern → A / B+C / D  (Step 3)
4. Copy a template          → templates/pattern-A-simple-click.spec.ts.template
5. Replace placeholders, save as src/test/PRValidation/PR<N><Comp>.spec.ts
6. cd sp-client/integration-tests/sp-pages-playwright && rushx playwright --grep "PR #<N>"
7. Read probe logs — AFTER must show prBuildCount > 0
8. Screenshots in temp/playwright/<BEFORE|AFTER>-pr<N>-<comp>-fullpage.png
```

---

## The 4 inputs you need before writing a spec

| Input | Where to get it |
|---|---|
| **PR number** | The PR you want to validate |
| **PR debug link** (PR_LOADER URL + PR_MANIFESTS URL) | ADO PR comment threads — see Step 1 |
| **Flight number(s)** | PR description "Visual verification" section, e.g. `1535`. Default for Wave-6 Panel→Drawer = `1535` |
| **Trigger selector + setup needed** | DOM selector that opens the changed UI, PLUS any data setup (comment, like, web part) required. See Step 3 + "Trigger pattern catalog" |

---

## Step 1 — Get PR debug link (10 sec)

A validation bot auto-posts a structured comment on every PR build containing the full debug link. Fetch via ADO REST:

```bash
REPO_ID=3829bdd7-1ab6-420c-a8ec-c30955da3205  # odsp-web
PR=<PR_NUMBER>
curl -sL -u ":$(az account get-access-token --query accessToken -o tsv)" \
  "https://onedrive.visualstudio.com/ODSP-Web/_apis/git/repositories/${REPO_ID}/pullRequests/${PR}/threads?api-version=7.0" \
  | grep -oE 'https://[^"]*odspwebcidev[^"]*sp-loader-assembly_default_[a-f0-9]+\.js' | head -1
```

Returns the full `PR_LOADER` URL. The matching `PR_MANIFESTS` is the same prefix without `-hashed/sp-loader-...js`, plus `/manifests.js`. Example:

```
PR_LOADER    = https://odspwebcidev.z13.web.core.windows.net/odsp-web-pr_2219557.002-hashed/sp-loader-assembly_default_57f796e701669261cf0455a8a945a437.js
PR_MANIFESTS = https://odspwebcidev.z13.web.core.windows.net/odsp-web-pr_2219557.002/manifests.js
```

**Do NOT** `curl <build>-hashed/` directory listing — Azure disabled directory indexing, returns 404.

**Fallback** (if bot comment missing): download `uploadMetadata` artifact (1.5 MB JSON) and `grep -oE 'sp-loader-assembly_default_[a-f0-9]+\.js'`.

---

## Step 2 — Pick flights

Most Wave-6 Panel→Drawer PRs need only **`1535`**.

| Flight | Name | Use when |
|---|---|---|
| **1535** | SPStableBundleEnabled | v9 OverlayDrawer migration PRs |
| 62874 | SPVisualRefreshEnabled | Large-icon AppBar |
| 61636 | NextGenSharePoint | NGSP client opt-in |
| 62743 | DeThemeSuiteNav | White SuiteNav |
| 62764 | SPSiteElevationEnabled | Canvas shadow |
| 60546 | DeThemeCanvasContent | New canvas theme |
| 61863 | SPFx debug flight gate | Required in addition if PR needs client `FeatureOverrides` cookies |

Multiple flights: comma-separate. Negative: `!N`.

---

## Step 3 — Identify trigger pattern

Match your PR to one of these patterns. Each has a working reference spec in `src/test/PRValidation/` — when writing a new spec, read the closest match as a starting point.

### Pattern A — Simple click on default SitePage
PR's UI is opened by clicking a button/icon that exists on EVERY published SitePage by default (social bar, command bar, page-level analytics, etc).

**Steps:** create page → publish → load with debug link → click trigger.

**Examples:**
- PR 2218733 AnalyticsPanel: `[data-automation-id="analyticsButton"]` (command bar)
- PR 2219541 LikesPanel: `[data-automation-id="sp-socialbar-likebutton"]` → `sp-socialbar-likedbymessage` (social bar)
- PR 2219557 PagePromotePanel: `[data-automation-id="promoteButton"]` (command bar)
- PR 2219568 BookmarkPanel: bookmark button → bookmarkmessage (social bar)

Easiest pattern. ~70 sec total.

### Pattern B — Requires data setup via REST
PR's UI shows only when there's data (e.g. a comment, a like, an item). Need to seed data via SharePoint REST API before triggering.

**Steps:** create page → publish → REST POST data → load with debug link → click trigger.

**Example:** PR 2219504 comment LikedByPanel — see also Pattern C below.

### Pattern C — Requires a SECOND user (multi-user)
PR's UI only shows when ANOTHER user performed an action (e.g. "X people liked YOUR comment" only renders if X excludes the current user).

The FIC synthetic tenant has THREE users by default:
- `adminUser` (default `rootSitePage` is logged in as this)
- `nonAdminUser`
- `globalReaderUser`

Open a page as the second user via `spPageProvider.loadPageAsync`:

```typescript
const nonAdminPage = await spPageProvider.loadPageAsync(publishedUrl, {
  user: nonAdminUser,
  forceManualLogin: true,
  disableSpClientDevAssets: true
});
// ... do REST or UI action as nonAdmin ...
await nonAdminPage.close();
```

**Example:** PR 2219504 — admin posts comment, nonAdmin likes it, then admin reloads to see "1 person liked your comment" link.

### Pattern D — Depends on an external product (try first, skip only after probing)
PR's UI is inside a 3rd-party web part or external product surface (Planner, Stream, Yammer, Viva Amplify, etc) that depends on a separate Microsoft 365 product. **Do not assume "external → skip"** — some external surfaces ARE registered on the synthetic tenant.

**Required first step: probe reachability before writing the full capture spec.** Write a short probe spec (~30-45 sec) that does one of:
- For web parts: open the web part picker and search for the dependency by name; check the result count.
- For app surfaces: `goto` the app's entry URL (e.g. `/_layouts/15/<app>.aspx`) and inspect title / hub chrome / access-denied signals.

**Outcome A — reachable:** Promote the probe to a full Pattern A/B/C capture spec (re-use the entry path the probe verified). Examples of surfaces that look Pattern D but are actually reachable on the synthetic tenant:
- Viva Amplify via `/_layouts/15/viva-amplify.aspx?path=Campaigns` (page title "Amplify", "Create a campaign" button present, no access-denied) — D3/D8/D9 drawer surfaces reachable from the campaign flow.

**Outcome B — confirmed unreachable:** Only after the probe gives clear negative signal (web part picker returns 0 results, app URL returns access-denied / 404 / redirect to a generic SP page) should the skill fall back to:
1. Skip — accept manual screenshot from PR author.
2. Ask the dependency owner to register the web part / app on the synthetic tenant.
3. Find a tenant with the dependency installed (rare for FIC-eligible tenants).

**Example of confirmed-unreachable:** PR 2219485 PlanCreationPanel — Planner web part picker search returns 0 results on the synthetic tenant; the spec captured this negative outcome in its diagnostic before falling back.

---

## Step 4 — Write the spec

Pick the closest reference spec from Step 3 as your starting point (e.g. copy `PR2219541LikesPanel.spec.ts` for Pattern A, `PR2219504LikedByPanel.spec.ts` for Pattern B+C). Replace these values:

| Replace | With |
|---|---|
| `PR_LOADER` | From Step 1 |
| `PR_MANIFESTS` | From Step 1 |
| `FLIGHTS` | From Step 2, e.g. `'1535'` |
| `<NUMBER>` | PR number — in const string, pageName, screenshot path |
| `<component>` | Short tag for filenames (e.g. `analytics`, `likes`, `promote`) |
| Trigger selector | DOM selector for the button/menu item |
| Test GUIDs (2) | Generate via `odsp-web:generate-guid` MCP tool. **NEVER hand-write GUIDs.** |
| owner | Your alias |

Save to `src/test/PRValidation/PR<NUMBER><Component>.spec.ts`.

---

## Step 5 — Run

```bash
cd sp-client/integration-tests/sp-pages-playwright
rushx playwright --grep "PR #<NUMBER>"
```

BEFORE + AFTER run in parallel via heft workers. ~70 sec for simple, ~120 sec for multi-user.

Output:
```
temp/playwright/<BEFORE|AFTER>-pr<NUMBER>-<component>-fullpage.png
```

Move to a permanent folder when done:
```bash
mkdir -p temp/playwright/PR-validation-final-v2
mv temp/playwright/*pr<NUMBER>*.png temp/playwright/PR-validation-final-v2/
```

---

## Step 6 — Verify

Read console output for each variant. Healthy signals:

| Field | BEFORE expected | AFTER expected | Meaning |
|---|---|---|---|
| `prBuildCount` | **0** | **100+** | Debug link loaded PR bundle |
| `hasV9Drawer` | true (v8 compat layer) | true | v9 OverlayDrawer rendered |
| `borderRadiusTopLeft` | `0px` | `16px` typically | v9 new rounded corners (most Wave-6 PRs) |
| `drawerWidth` | varies | varies | Sometimes same as v8 (means no visual regression) |

**If AFTER `prBuildCount=0`:** Allow-button regex missed the localized text. Add the actual button text to the regex.

**If BEFORE/AFTER look identical:** the PR might be visually equivalent on purpose (e.g. PR 2219557 — v9 with default `size='medium'` ≈ v8 default). The `prBuildCount` probe proves PR build loaded; that's the technical proof. Note this in PR description.

---

## Critical gotchas (do not skip)

1. **URL parameter names matter.** `enableFeatures=N` and `pseudo=true` are silently ignored. Use **`debugFlights=N`**. Source: `sp-client/libraries/sp-tab-tasklib/src/SPTaskLib/PageUtil.ts`. **Do NOT add `market=qps-ploc`** — it renders pseudo-localized text (Ĺōàď ďēb...) that clutters AFTER screenshots. The technical proof that the PR build loaded is the `prBuildCount > 0` console probe, not visual pseudo-loc. If you really need a quick visual sanity check during local dev, add it temporarily but strip it before committing the spec.

2. **v9 OverlayDrawer also gets `.ms-Panel-main`** as a compat class. Selecting only by `.ms-Panel-main` matches both v8 and v9. The authoritative v9-only selector is `[class*="fui-OverlayDrawer"]:not([class*="__backdrop"])`.

3. **`-hashed/` vs non-hashed.** Loader needs `-hashed`. Manifests does NOT.

4. **PR assertions decay.** When AFTER fails on backdrop/padding/icon assertion months after writing, re-read PR's latest commits. The spec is usually the stale one.

5. **Don't use `_spPageContextInfo` to read NGSP state on dogfood SitePage** — it's been renamed. Read `window.__ngsp_IsNextGenSharePointExperienceOptedIn` instead. (Not relevant for synthetic-tenant FIC tests, but useful for dogfood debugging.)

6. **Synthetic tenant ≠ NGSP-opted-in.** New chrome (white SuiteNav, canvas shadow, large-icon AppBar) is NOT visible on the synthetic tenant — that requires server-side NGSP opt-in which only some dogfood/EAP tenants have, none of which support FIC. **Hard architectural limit.** Do not try to spoof `__ngsp_*` on the client — proven not to work. For full new-chrome reference, open a dogfood site manually in your browser.

---

## Working spec examples by pattern

| Pattern | Example spec | Key feature |
|---|---|---|
| A (simple click) | `PR2219541LikesPanel.spec.ts` | Single trigger selector |
| A | `PR2219557PagePromotePanel.spec.ts` | Command-bar trigger |
| A | `PR2219568BookmarkPanel.spec.ts` | Social-bar trigger |
| A (multi-flight) | `PR2218733AnalyticsPanel.spec.ts` | 6 flights combined |
| B+C (REST + multi-user) | `PR2219504LikedByPanel.spec.ts` | admin posts comment, nonAdmin likes |
| D (probe first, escalate only if unreachable) | — | PR 2219485 PlanCreation: probe Planner picker → 0 results → skipped; PR 2225561 Amplify drawers: probe `/_layouts/15/viva-amplify.aspx` → 200 + "Create a campaign" present → capture proceeds via campaign flow |

---

## Where everything lives

| | Path |
|---|---|
| Project root | `sp-client/integration-tests/sp-pages-playwright` |
| Specs | `src/test/PRValidation/PR<NUMBER><Component>.spec.ts` |
| Screenshots (raw) | `temp/playwright/<BEFORE|AFTER>-pr<NUMBER>-<component>-fullpage.png` |
| Screenshots (curated) | `temp/playwright/PR-validation-final-v2/` |
| Workflow doc (extended) | `src/test/PRValidation/docs/PR-validation-FIC-workflow.md` |
