# Shipping Announcement Agent

Home for the **shipping-announcement** Claude Code skill so it can run as an hourly **cloud routine** (which needs a git-checked-out copy + durable state it can commit back).

## What it does
Each run sweeps two sources and DMs the relevant **Primary CSM** a ready-to-paste, CSM-voiced customer announcement (never posts to a customer channel):
- **Source A — feature ships:** new customer-facing posts in the `#shipped` Slack channel.
- **Source B — bug fixes:** newly-completed Production-Support tickets in Linear.

It is **incremental + idempotent**: it only acts on items newer than the watermark in `shipping-announcement-state.json`, so a run with nothing new sends nothing.

## Files
- `.claude/skills/shipping-announcement/SKILL.md` — the full agent procedure (source of truth).
- `shipping-announcement-state.json` — the durable watermark + caches. **The cloud routine reads this at start and commits + pushes it back at end** — that is what prevents duplicate sends.
- `channel-mapping-export.json` — customer → Slack channel map (for the paste-target link).

## CSM routing
Uses the Salesforce `New_Primary_CSM__c` / `New_Secondary_CSM__c` fields (NOT the deprecated `Primary_CSM__c` / `Secondary_CSM__c`).

## Connectors the cloud routine needs
Slack, Linear, and CData Connect AI (Salesforce).
