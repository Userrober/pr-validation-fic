# pr-validation-fic

Capture BEFORE/AFTER screenshots for SharePoint client PRs on the FIC synthetic tenant.

## What it does

Give Claude a PR number — it reads the PR description and changed files, decides what trigger
pattern fits (simple click, REST setup, multi-user, external dependency), writes a custom
Playwright spec at `sp-client/integration-tests/sp-pages-playwright/src/test/PRValidation/`,
runs it twice (once against prod CDN for BEFORE, once with the PR build's debug link for
AFTER), and produces two full-page screenshots ready to drop into the PR description.

Tenant state is mutated when needed (page creation, list-setting toggles, REST seeding,
second-user actions) and cleaned up in a `finally` block so the shared synthetic tenant
stays clean for the next run.

## Install

In your Claude Code project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "pr-validation-fic-marketplace": {
      "source": {
        "source": "github",
        "repo": "Userrober/pr-validation-fic"
      }
    }
  },
  "enabledPlugins": {
    "pr-validation-fic@pr-validation-fic-marketplace": true
  }
}
```

Restart Claude Code. The `pr-validation-fic` skill becomes available.

## Prerequisites

- `odsp-web` repo with `rush install` complete
- `az login` against the Microsoft tenant (so the skill can fetch the PR debug link from ADO)
- VPN / network access to `odspwebcidev.z13.web.core.windows.net` (PR build CDN)

## Usage

In Claude Code, just say:

> validate PR 2219541 with pr-validation-fic

Claude loads the skill, walks the 6 steps, drops screenshots into
`temp/playwright/PR-validation-final-v2/`, and tells you the markdown
attachment snippet to paste into the PR description.

## How it relates to other validation tools

This skill is the **per-PR custom-spec capture path**. It is intentionally allowed to
mutate tenant state (list settings, page creation, REST seeding) — which means it can
capture surfaces that fixed-page automation cannot. See the SKILL.md "What this skill
actually does" section for the full contract.

## Layout

```
pr-validation-fic-plugin/
  .claude-plugin/
    plugin.json
  skills/
    pr-validation-fic/
      SKILL.md
  README.md
```
