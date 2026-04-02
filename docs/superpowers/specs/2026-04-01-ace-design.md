# ACE (AI Connect Engine) — Design Spec

## Overview

ACE is a Claude Code plugin that orchestrates the CRISPR-Connect lifecycle — taking an idea through app building, Connect setup, LLO management, and closeout. It operates as an AI "mid-level operator" of a Connect opportunity, automating the multi-step process flow designed by Matt (Managing Director of Connect).

ACE follows the same architecture as the canopy plugin: agents orchestrate skills, skills are prompt-based capability definitions, and MCP servers provide programmatic access to external systems.

## Goals

1. **Living operational playbook** — ACE's plugin structure IS the plan. The agent definitions, skill files, and integration docs are both the system's instructions and the team's documentation.
2. **Two execution modes** — "auto" (runs through, emails admin group at checkpoints) and "review" (pauses at gates for human approval). Both modes execute the same steps.
3. **Self-improving** — skills improve by editing the SKILL.md and pushing to the plugin repo. Same pattern as canopy.
4. **Individually replaceable components** — each skill handles one step. Improving ACE means improving one skill at a time.
5. **Generated documentation** — `/ace:docs` generates a human-readable playbook from agent and skill definitions, always in sync with the actual system.

## Architecture

### Plugin Structure

```
ace/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   ├── ace-orchestrator.md          # Top-level: dispatches to phase agents
│   ├── app-builder.md               # IDD → Nova → test → deploy to CommCareHQ
│   ├── connect-setup.md             # Workspace → Program → Opportunity → config → invites
│   ├── llo-manager.md               # Onboarding → OCS → monitoring
│   └── closeout.md                  # Invoicing → feedback → learnings → grading
├── commands/
│   ├── run.md                       # /ace:run <opp-id> [--mode auto|review]
│   ├── step.md                      # /ace:step <step-name> <opp-id>
│   ├── status.md                    # /ace:status [opp-id]
│   └── docs.md                      # /ace:docs — generate playbook from source
├── skills/
│   ├── idea-to-idd/SKILL.md
│   ├── idd-to-learn-app/SKILL.md
│   ├── idd-to-deliver-app/SKILL.md
│   ├── app-test/SKILL.md
│   ├── app-deploy/SKILL.md
│   ├── training-materials/SKILL.md
│   ├── connect-program-setup/SKILL.md
│   ├── connect-opp-setup/SKILL.md
│   ├── llo-invite/SKILL.md
│   ├── llo-onboarding/SKILL.md
│   ├── llo-uat/SKILL.md
│   ├── llo-launch/SKILL.md
│   ├── ocs-agent-setup/SKILL.md
│   ├── timeline-monitor/SKILL.md
│   ├── flw-data-review/SKILL.md
│   ├── opp-closeout/SKILL.md
│   ├── llo-feedback/SKILL.md
│   ├── learnings-summary/SKILL.md
│   └── cycle-grade/SKILL.md
├── playbook/
│   └── integrations/
│       ├── connect-api.md           # Connect API requirements + gaps
│       ├── commcare-api.md          # CommCare API requirements + gaps
│       ├── ocs-integration.md       # OCS agent management + transcript access
│       └── nova-integration.md      # Nova fork/API requirements
├── mcp/
│   ├── google-drive-server.ts       # Google Drive + Sheets MCP (built)
│   └── ocs-server.ts                # OCS MCP (to build)
├── templates/                       # Email, IDD, training, collateral templates
├── scripts/                         # Utility scripts for skills
└── docs/
    └── generated/
        └── playbook.md              # Auto-generated from /ace:docs
```

### Agents

The top-level ace-orchestrator dispatches to 4 phase agents. Each phase agent knows its step ordering, dependencies, gate behavior, and mode logic (auto vs. review). Workflow logic lives in the agent definitions, not in a separate manifest.

| Agent | Skills it orchestrates | Primary owner |
|-------|----------------------|---------------|
| **ace-orchestrator** | Dispatches to phase agents, tracks overall opp state | Jon |
| **app-builder** | idea-to-idd, idd-to-learn-app, idd-to-deliver-app, app-test, app-deploy, training-materials | Cal + Neal |
| **connect-setup** | connect-program-setup, connect-opp-setup, llo-invite | Cal |
| **llo-manager** | llo-onboarding, llo-uat, ocs-agent-setup, timeline-monitor, flw-data-review | Jon |
| **closeout** | opp-closeout, llo-feedback, learnings-summary, cycle-grade | Jon |

The agent/skill boundary is expected to evolve. We may collapse or split agents as we test. The current grouping reflects natural phases of the process.

### Skills (22)

Each skill is a SKILL.md file that handles one step of the CRISPR-Connect process. Skills are stateless prompt definitions that Claude executes. They read from and write to the opportunity's Google Drive folder and call MCP tools for external system access.

**Process flow with skill mapping:**

| # | Step | Skill | LLM Judge | Notes |
|---|------|-------|-----------|-------|
| 1 | Establish contracts with LLOs | (manual — Sarvesh) | | Not automated |
| 2 | Idea generated | (trigger) | | Neal generates initial ideas |
| 3 | AI-supported concept iteration → IDD | `idea-to-idd` | Yes | Neal upskills Shakes for this |
| 4 | Pass IDD to Nova → Learn app | `idd-to-learn-app` | Yes | Depends on Nova bot API |
| 5 | Pass IDD to Nova → Deliver app | `idd-to-deliver-app` | Yes | Can run parallel with step 4 |
| 6 | Automated test plan & execution | `app-test` | Yes | Able to take Nova Learn or Deliver app output, test, and provide clear guidance back to Nova to improve application. Based on GS team's work |
| 7 | Upload apps to CCHQ, build & publish | `app-deploy` | | Requires new CommCareHQ API, which could also be leveraged by Nova. |
| 8 | Create training materials | `training-materials` | Yes | Can run parallel with step 7.  Option to leverage Clayton's video generation pipeline. |
| 9 | Create/configure Program in Connect | `connect-program-setup` | | Needs Program API (CCC-301) |
| 10 | Create/configure Opportunity | `connect-opp-setup` | | Needs Opportunity API (CCC-301) |
| 11 | Configure verification rules, units, payments | `connect-opp-setup` | | Folded into opp setup — may split into own skill if complex |
| 12 | Look up LLO contacts, send invites | `llo-invite` | | Needs Invite API + LLO Directory |
| 13 | Email LLOs with training + instructions | `llo-onboarding` | | Via Ace-AI@Dimagi.com |
| 14 | Lead LLO through User Acceptance Testing (UAT) of the Learn and Deliver applications. | 'llo-uat' | Yes | Gather feedback needs improvement prior to use.  Able to go back to idd-to-learn or idd-to-deliver to improve application |
| 15 | Lead LLO to Launch application  | `llo-launch` |  | After successful UAT |
| 16 | OCS answer bot for LLO questions | `ocs-agent-setup` | Yes | ACE configures OCS agent per opp |
| 17 | Monitor LLO timeline compliance | `timeline-monitor` | Yes | Recurring during active opp |
| 18 | Review FLW data, suggest improvements | `flw-data-review` | Yes | Recurring during active opp |
| 19 | Pull invoices, create Jira payment ticket | `opp-closeout` | | Triggered at opp end date |
| 20 | Prompt LLO for feedback | `llo-feedback` | | |
| 21 | Summarize learnings, create new IDD | `learnings-summary` | | Can trigger another cycle |
| 22 | Grade overall CRISPR-Connect cycle | `cycle-grade` | Yes | |

### MCP Servers

**In ACE repo:**
- **Google Drive MCP** (`mcp/google-drive-server.ts`) — Sheets + Drive tools. Already built. Service account: `gws-local-dev@dimagi-chrome-extension.iam.gserviceaccount.com`. This is the canonical Google Drive MCP, registered in canopy's registry for cross-project use.
- **OCS MCP** (`mcp/ocs-server.ts`) — To build. Agent management (create/configure agents per opportunity), transcript access (read LLO conversations for analysis), context injection.

**In connect-labs (external):**
- **commcare-hq MCP** — 26 tools. CommCare app structure, Connect solicitations/reviews/awards/funds, Google Sheets reading. Production-ready.
- Needs extension with: Program/Opportunity CRUD (CCC-301), invite API, verification rules, delivery units, payment config, invoice pull.

**Not yet determined:**
- **Nova** — May need a fork to enable bot-controlled app generation. Needs exploration with Braxton.

### Opportunity State (Google Drive)

Each opportunity ACE manages gets a folder in Google Drive:

```
ACE/
  <opp-name>/
    state.yaml              # Current step, mode, timestamps, gate approvals
    idd.md                  # Intervention Design Doc
    app-summaries/
      learn-app-summary.md
      deliver-app-summary.md
    test-results/
    training-materials/
    comms-log/
    closeout/
```

The team can open any opportunity folder and see exactly where things stand. ACE reads/writes this programmatically via the Google Drive MCP.

### Execution Modes

**Auto mode:** ACE runs through all steps without pausing. Emails the CRISPR Admin Dimagi Google Group (Neal, Jon, Matt, Sarvesh, Cal) at each step completion and on failures. Gates are logged but not enforced.

**Review mode:** ACE pauses at gate steps and waits for human approval before proceeding. Gates are defined in each agent's logic. Key gates:
- IDD approval (before building apps)
- App build (to iterate on questions Nova prompts about application creation)
- App deployment approval (before publishing to CCHQ to review application and test results)
- LLO invite approval (before sending invitations)
- Post LLO UAT (determine if need to rework applications prior to launching)

### LLM-as-Judge

Many skills have LLM-as-Judge evaluation. This means ACE evaluates the quality of its own output at that step before proceeding. The criteria are defined within each skill's SKILL.md. Over time, these evaluations improve as skills are refined.

### OCS Integration

OCS is ACE's "mouth and ears" for LLO interaction. ACE manages OCS — it creates and configures an OCS agent per opportunity, injecting the IDD, training materials, and opportunity context. LLO questions come in via Ace-AI@Dimagi.com, are handled by OCS, and ACE monitors the transcripts via OCS APIs (already exist) to analyze sentiment, identify issues, and recommend next steps. All OCS responses are cc'd to the admin group for monitoring.

### Video Generation Integration

Clayton has been working on a pipeline to create local videos for training and in-app Learn content.  As part of the training-materials skill, we could optionally leverage it to create training videos.

## Open Questions

1. How do we leverage CommCareHQ domains for app-deploy?  The current Connect model of one domain per LLO seems more onerous that necessary.  Unclear if we can get away with just a single 'crispr-connect' domain with preset feature flags and data forwarding setup.
2. How are we setting up Programs and Workspaces in Connect for this?  Currency is currently defined at the Program level, so one Program may not be able to cover Proof of Concept's in multiple countries without creating multiple programs.
3. How much do the Connect API's need to expose in the configuration?  Are there defaults that can be used that don't require the API's?  For example, verification rule defaults vs configuring verification rules via API.

## Technical Enablement (Parallel Track)

These are the APIs and platform changes developers need to build. They unblock specific ACE skills.

| Priority | What | Where | Ticket | Blocks | Who |
|----------|------|-------|--------|--------|--------|
| 1 | REST API for Program + Opportunity creation | Connect backend + connect-labs MCP | CCC-301 | connect-program-setup, connect-opp-setup | Cal |
| 2 | New generic delivery type ("CRISPR-Connect") | Connect backend | — | connect-opp-setup | Cal |
| 3 | CommCareHQ API to upload JSON app and build/publish it | CommCareHQ backend | — | app-deploy | Cal |
| 4 | LLO Directory in proper data model/DB | Connect backend | — | llo-invite | Cal |
| 5 | v2 LLO Entity implementation | Connect backend | CCC-300 | llo-invite, llo-onboarding | Cal |
| 6 | Invoice pull + Jira ticket creation | Connect backend | — | opp-closeout | Cal |
| 7 | Verification rules / delivery units / payment config API | Connect backend + connect-labs MCP | — | connect-opp-setup | Cal |
| 8 | Invite API | Connect backend + connect-labs MCP | — | llo-invite | Cal |
| — | Combine Learn + Deliver into single HQ app | CommCare | — | (nice to have) | Cal |
| — | connect-cli | — | — | (nice to have) | Cal |
| — | Pull in Clayton's video generation work | — | training-materials | (nice to have) | Clayton |

Skills that depend on unbuilt APIs start as stubs with manual fallback instructions.

## Improvement Model

Same as canopy. Skills improve by editing the SKILL.md and pushing to the plugin repo. The goal is continuous self-improvement:

1. Run a skill on a real opportunity
2. Evaluate the output (LLM-as-Judge + human review)
3. Identify what went wrong or could be better
4. Edit the SKILL.md with better instructions, examples, or tool usage
5. Push to the ACE plugin repo
6. Next run uses the improved skill

No formal interface contracts or versioning beyond what git provides. The agent definitions and skill files are the source of truth.

## Generated Documentation

`/ace:docs` reads all agent definitions and skill files and generates a single human-readable playbook at `docs/generated/playbook.md`. This document:
- Explains the full CRISPR-Connect process flow
- Documents what each agent and skill does
- Shows the dependency graph between steps
- Lists gate points and mode behavior
- Describes MCP integrations and what APIs are needed
- Is always in sync with the actual system because it's generated from the source

## Key Stakeholders

| Person | Role | Owns |
|--------|------|------|
| Matt | Managing Director of Connect, initiative owner | Process flow, business requirements, project management |
| Neal | Idea generation, IDD creation | idea-to-idd skill, idea fountain |
| Cal | Connect tech lead | app-builder agent, connect-setup agent, Connect APIs |
| Sarvesh | LLO contracts, Auto-Connect | LLO relationships, testing |
| Jon | ACE agent development | llo-manager agent, closeout agent, MCP servers, overall architecture |

## Source Reference

Planning spreadsheet: https://docs.google.com/spreadsheets/d/1XxcPxK1oYtDxcfmElBb73U2UtLYEodiaMUazjmEVAWE/edit
- Tab "Process Flow" — 21-step process with owners, skills, and next steps
- Tab "Connect Tech Work" — Prioritized API/platform work for Cal's team
