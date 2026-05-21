# pr-validation-fic

A Claude Code plugin that captures **BEFORE/AFTER** screenshots for SharePoint client PRs on Microsoft's internal FIC synthetic tenant. Claude reads the PR description, picks the right trigger pattern, writes a custom Playwright spec, runs it twice (prod CDN vs PR build via debug link), and gives you two full-page screenshots ready to drop into the PR description.

> **Audience:** this plugin is built for engineers working in the Microsoft `onedrive/odsp-web` monorepo. It will not do anything useful outside that repo.

## One-line install

```bash
claude plugin marketplace add Userrober/pr-validation-fic && claude plugin install pr-validation-fic@pr-validation-fic-marketplace
```

Restart Claude Code after installing. The `pr-validation-fic` skill becomes available — invoke it by mentioning a PR number, e.g.:

> validate PR 2219541 with pr-validation-fic

## What it actually does

When invoked, Claude:

- Reads the PR description + changed files to infer where the UI lives and what setup it needs
- Picks the right reference pattern (simple click / REST setup / multi-user / external dependency)
- Writes a brand-new Playwright spec per PR at `sp-client/integration-tests/sp-pages-playwright/src/test/PRValidation/`
- Mutates tenant state when needed (creates pages, toggles list moderation, posts REST data, drives a second user, etc.) and **self-cleans in a `finally` block**
- Runs the spec, reads probe logs, captures BEFORE/AFTER screenshots, moves them to a curated folder
- Hands you the markdown attachment snippet to paste into the PR description

Surfaces that need list-setting changes, multi-user setup, or REST-seeded data are in scope. Even Pattern D (external web-part dependency) is **probed first** — only after a quick reachability check proves the dependency truly isn't registered on the synthetic tenant does it fall back to manual handoff.

## Prerequisites

You'll need a working `odsp-web` checkout:

- The `onedrive/odsp-web` repo (access required)
- `rush install` has completed at the repo root
- `az login` against the Microsoft tenant — the skill needs `az account get-access-token` to fetch the PR's debug link from ADO
- Network access to the internal PR build CDN (corp VPN usually fine)

The synthetic tenant itself is shared and pre-provisioned — no per-user setup.

## Usage

After install, in any Claude Code session inside the `odsp-web` repo:

```
validate PR <number> with pr-validation-fic
```

Claude walks the 6 steps documented in `skills/pr-validation-fic/SKILL.md`:

1. Fetch PR debug link from ADO
2. Pick flights (usually just `1535` for Wave-6 Panel→Drawer PRs)
3. Identify trigger pattern (A / B / C / D)
4. Write the spec
5. Run BEFORE + AFTER in parallel
6. Verify probe signals + collect screenshots

Output lands at `sp-client/integration-tests/sp-pages-playwright/temp/playwright/PR-validation-final-v2/`.

## Uninstall

```bash
claude plugin uninstall pr-validation-fic@pr-validation-fic-marketplace
claude plugin marketplace remove pr-validation-fic-marketplace
```

## Layout

```
.claude-plugin/
  marketplace.json   # single-plugin marketplace wrapper
  plugin.json        # plugin metadata
skills/
  pr-validation-fic/
    SKILL.md         # the full skill content
README.md
```

## Source of truth

The canonical SKILL.md lives in the `odsp-web` repo at `.ai/pr-validation-fic-plugin/skills/pr-validation-fic/SKILL.md`. This GitHub repo is a published mirror so Claude Code can fetch it as a marketplace.
