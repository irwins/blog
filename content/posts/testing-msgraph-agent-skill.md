+++
title = "Testing the msgraph Copilot Skill: Giving AI Agents a Local Graph API Brain"
date = 2026-03-05
draft = false
tags = ["Microsoft Graph", "GitHub Copilot", "AI Skills", "Entra ID", "PIM", "Global Secure Access"]
author = "Irwin Strachan"
+++

## The Problem I Wanted to Solve

Large language models are trained on data that is months — sometimes over a year — old. The Microsoft Graph API has **27,700+ endpoints** and is **updated weekly**. That gap creates a real problem: AI agents hallucinate endpoints that don't exist, suggest deprecated paths, miss required permissions, or get `$filter` syntax wrong.

I wanted to see whether the open-source [`msgraph` skill](https://github.com/merill/msgraph) by Merill Fernando could close that gap — and whether I could wire it directly into specialist Copilot agents for a fully grounded, anti-hallucination workflow.

In my case, this wasn't theoretical. I hit one run where Copilot suggested a stale Graph path, then another where the path looked fine but permissions still failed at execution. That was enough for me to stop trusting best-guess output and move to a lookup-first workflow.

I learned this the hard way in my earlier PIM access review automation work: I almost optimized the wrong workflow because I solved what looked technically interesting, not what the reviewers actually needed. I do not want to repeat that mistake with Graph automation.

## Agent Files

If you want to use the same specialist agent setup, download the exact files here:

- [Download `graph-analyst.agent.md`](/blog/downloads/testing-msgraph-agent-skill/graph-analyst.agent.md)
- [Download `graph-reviewer.agent.md`](/blog/downloads/testing-msgraph-agent-skill/graph-reviewer.agent.md)

## Why This Matters to Me

This wasn't my first MgGraph automation project. In my earlier post on [PIM access review](../../posts/pim-access-review/), I worked through the practical edge cases of Entra role access review automation and learned the hard way where Graph workflows break down in real environments. That background is exactly why I built this lookup-first approach: I wanted reliability over guesswork.

## Why I Added the `msgraph` Skill to My Toolkit

The skill bundles the complete Microsoft Graph API surface as **local indexes** that an agent can search instantly — no network calls, no latency. It ships as a set of pre-compiled CLI binaries (macOS, Linux, Windows) with four searchable indexes:

| Index | Count | What It Contains |
|---|---|---|
| OpenAPI endpoints | 27,700+ | Path, method, summary, description, permission scopes |
| Endpoint docs | 6,200+ | Permissions (delegated/app), query parameters, required headers |
| Resource schemas | 4,200+ | Properties with types, supported `$filter` operators |
| Community samples | Growing | Hand-verified queries mapping natural-language tasks to exact API calls |

Before I trusted it, I needed three things: endpoint coverage, permission accuracy, and speed during live investigation. The local index model checked all three in my workflow. I still use it as a **knowledge layer**, not an execution engine, and pair it with my scripts for actual API calls.

## Setting Up the Skill in My Repo

I integrated the skill into an existing Azure DevOps repo focused on Entra ID PIM automation. The directory structure:

```
.agents/
└── skills/
    └── msgraph/
        ├── SKILL.md              # Skill manifest and instructions
        ├── CONTRIBUTING.md       # How to add community samples
        ├── references/
        │   ├── REFERENCE.md      # Quick-reference for common patterns
        │   ├── graph-api-index.json      # 27,700+ OpenAPI endpoints
        │   ├── api-docs-index.json       # 6,200+ endpoint docs
        │   ├── samples-index.json        # Community samples
        │   └── docs/             # Concept docs (paging, batching, etc.)
        └── scripts/
            ├── run.sh            # macOS/Linux launcher
            ├── run.ps1           # Windows launcher
            └── bin/              # Pre-compiled binaries per platform
```

I then created two specialist agents in `.github/agents/` and hardwired them to use the skill by default:

- **`graph-analyst`** — Designs accurate Graph API calls following a progressive lookup strategy.
- **`graph-reviewer`** — Reviews Graph scripts for correctness, security, and maintainability.

Both agents have the same first instruction: *"Always use the local `msgraph` skill as the first source of truth."*

I prefer shipping these as blog assets so the post and file versions stay in sync. A gist is still useful as an optional mirror for sharing.

## Test 1: Finding the `accessReview` Endpoint

**Prompt:** *"What is the endpoint for accessReview?"*

The agent ran `api-docs-search` and `openapi-search` and returned both:

- **Modern path:** `GET /identityGovernance/accessReviews/definitions` (v1.0)
- **Deprecated path:** `GET /accessReviews` (beta, deprecated)

It flagged the deprecation and recommended the modern path. Without the skill, an LLM would likely have returned the deprecated endpoint — or worse, invented one.

My read: this was a strong start for endpoint discovery, especially where v1.0 and beta history overlap.

## Test 2: PIM for Groups Endpoints

**Prompt:** *"What are the PIM for Groups endpoints?"*

The agent ran:

```bash
bash .agents/skills/msgraph/scripts/run.sh api-docs-search \
  --endpoint "/identityGovernance/privilegedAccess/group" --method GET
```

It returned the full family of endpoints:

| Endpoint | Method | Purpose |
|---|---|---|
| `/identityGovernance/privilegedAccess/group/assignmentScheduleRequests` | POST | Request a new PIM group assignment |
| `/identityGovernance/privilegedAccess/group/assignmentScheduleInstances` | GET | List active assignment instances |
| `/identityGovernance/privilegedAccess/group/eligibilityScheduleRequests` | POST | Request eligibility for a group role |
| `/identityGovernance/privilegedAccess/group/eligibilityScheduleInstances` | GET | List active eligibility instances |

It also provided the required permission: `PrivilegedAssignmentSchedule.ReadWrite.AzureADGroup`.

My read: this saved the most time. Scope guidance in the same lookup removed a lot of trial-and-error.

## Test 3: Creating a PIM Assignment Request

**Prompt:** *"Show me how to create an assignmentScheduleRequest."*

The agent ran:

```bash
bash .agents/skills/msgraph/scripts/run.sh api-docs-search \
  --resource privilegedAccessGroupAssignmentScheduleRequest --method POST
```

It returned the full resource schema, and produced a ready-to-run PowerShell example:

```powershell
$requestBody = @{
    accessId     = "member"
    principalId  = "<user-or-group-object-id>"
    groupId      = "<target-group-id>"
    action       = "adminAssign"
    scheduleInfo = @{
        startDateTime = (Get-Date).ToUniversalTime().ToString("o")
        expiration    = @{
            type     = "afterDuration"
            duration = "PT8H"
        }
    }
    justification = "Deploying change CHANGE-1234"
} | ConvertTo-Json -Depth 5

Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/v1.0/identityGovernance/privilegedAccess/group/assignmentScheduleRequests" `
    -Body $requestBody `
    -ContentType "application/json"
```

This mattered because I standardize on `Invoke-MgGraphRequest` in my automation scripts. It correctly used that cmdlet instead of the non-existent `Invoke-MgGraphRestMethod` that LLMs often hallucinate.

My read: high confidence for schema and request construction, but I still validate production-bound payloads through dry runs.

## Test 4: Microsoft Entra Private Access

The agent ran multiple `openapi-search` queries and mapped out the full `networkAccess` resource tree:

- **Forwarding Profiles:** `POST /beta/networkAccess/forwardingProfiles`
- **Filtering Policies:** `POST /beta/networkAccess/filteringPolicies`
- **Policy Rules:** `POST /beta/networkAccess/filteringPolicies/{id}/policyRules`
- **Branch Connectivity:** `POST /beta/networkAccess/connectivity/branches`
- **Settings:** `PATCH /beta/networkAccess/settings/conditionalAccess`

It flagged that all of these are `beta`-only and noted the required permission: `NetworkAccess.ReadWrite.All`.

This is an area where generic LLM knowledge is thin. The skill's local OpenAPI index made the difference.

My read: very useful for newer Entra surfaces where generic model knowledge is still patchy.

What I did next: I treated these as design-time endpoints only, documented the beta constraint, and avoided wiring production automation until we had a stable rollout path.

## Where I Still Step In

Once I had the right endpoint and permission, I validated the role context and tenant state manually before execution. The lookup layer got me to the right API shape quickly. The [cli](https://graph.pm/reference/cli/#authentication-commands) does support authentication, something I'll look into in the near future.

That boundary is important for me. Fast local lookup lowers hallucination risk, but live Graph writes still need operational guardrails. For now I'd rather be in the driver's seat...

## What Worked Well

1. **Anti-hallucination by design.** The progressive lookup strategy (`sample-search` → `api-docs-search` → `openapi-search`) grounds answers in indexed data.

2. **Permission and deprecation accuracy.** Getting scopes and deprecation cues in one pass helped me avoid bad defaults.

3. **Offline-first speed.** Local indexes gave near-instant lookups with no network dependency and no token cost.

Hardwiring this behavior into `.agent.md` files also reduced repeated prompt setup. Each session starts with the same lookup discipline.

## What I Trust vs What I Still Verify

I trust the skill for endpoint discovery, permission direction, and rapid lookup during investigations.

I still verify three things manually before any write path: tenant context, role assignment blast radius, and rollback options.

Two constraints are worth calling out:

1. **Skill updates.** The indexes are static snapshots. When Microsoft ships new endpoints, you need a new skill release. The `skills-lock.json` tracks the version, but there's no auto-update mechanism yet. Fast local lookup is the upside; snapshot staleness is the trade-off.
2. **Community samples are sparse.** The curated sample library is growing but still small (16 samples as of 2026-03-05). Contributing your own working queries back is the practical way to improve this over time.

## My Setup in a Nutshell

```
.github/
├── copilot-instructions.md          # Repo-wide rules (Graph, commits)
└── agents/
    ├── graph-analyst.agent.md       # Endpoint design specialist
    └── graph-reviewer.agent.md      # Code review specialist

.agents/
└── skills/
    └── msgraph/                     # Local Graph API knowledge base
```

## My Verdict (For Ops Teams)

If you run Graph-heavy ops automation, adopt this now as a lookup layer. It turns Copilot from "probably right" into "show me the endpoint and scope," which is a much safer baseline.

My non-negotiable rule: I do not ship write-capable Graph automation that has not passed lookup validation plus environment-specific safety checks.

If your team needs end-to-end execution guarantees out of the box, wait until you define guardrails for tenant validation and change control. The skill improves answer quality, but it does not replace operational controls.

For me, pairing it with specialist agents that always look before they leap is already catching deprecated paths and permission mismatches that used to slip through.

## Shout out to Merill

Credit where credit is due: Merill Fernando built and maintains a tool that makes real-world Graph work safer for the rest of us. This isn't just "nice for demos" tooling. It has practical, day-to-day value when you're trying to avoid bad API calls, bad scopes, and bad assumptions.

If you work with Microsoft Graph and have not explored it yet, start here: [graph.pm](https://graph.pm/)

Open source only works long-term when people contribute back. If this tool saves you time, send a PR, add a sample, or sponsor the work.

## If You Want to Try This in 30 Minutes

1. Install the `msgraph` skill locally and confirm the binaries run on your platform.
2. Run one endpoint lookup for a task you already automate, then verify scope output against your current script.
3. Add one agent instruction to always run lookup before code generation, then compare output quality in two back-to-back prompts.

Build fast, validate hard, ship safely.

Ttyl,

Urv
