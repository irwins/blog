+++
title = 'Agent Ready Platform Repo'
date = 2026-02-02T18:50:18+01:00
draft = false
tags = ['AI', 'GitHub Copilot', 'Platform Engineering', 'Bicep', 'Azure', 'IaC']
+++

Hi! It's been a minute! Been trying to keep up with the latest trends in AI. Honestly, it's almost a full-time job.

I've been following what the capabilities are for quite some time now. I remember when Prompt engineer was the next thing, then came context engineering. Then we had MCP servers and agents... whole fleets!

Agents really changed the game! The latest addition is skills. Skills seem very promising... Here's where my experiment led me...

Oh, full disclosure I did use AI to help me out with the blog post... anyway, here's what I've been working on.

## Why "Agent-Ready" matters

In Platform Engineering, we usually focus on automation for people or CI/CD pipelines. But as I started using AI more in my own workflow, I realized that "standard" automation just isn't enough. We need to build repositories that aren't just machine-readable, but actually ready for an AI agent to navigate.

So, I’ve been working on transforming a standard Azure IaC repository into what I call an "agent-ready" environment.

## Closing the gap between us and the agent

Standard repos are built for humans, and humans are surprisingly good at dealing with ambiguity. We read between the lines, we remember that one Slack message about naming conventions, and we just "know" why we use certain Azure modules.

But Copilot and other AI agents don't have that context. They suffer from what I call the "Agent Context Gap"—the space between the code they see and the unspoken rules we have in our heads.

The goal here isn't just better autocomplete. I want the AI to be a collaborator that actually understands the project. By giving it a structured set of instructions and tools (skills), we can turn an agent into a junior engineer that not only knows how to write Bicep code but understands why we use Azure Verified Modules (AVM) and where our validation logic lives.

## The Architecture of an Agent-Ready Repo

In the `platform-agent-kit` repo, I followed a specific scaffold designed for AI interaction. But more important than the folder structure is the reasoning framework behind it.

![Agent-Ready Architecture](../../images/agent-ready-platform-repo/architecture.drawio.png)

*Figure 1: The dependency flow from mandatory context (.agents.md) to guardrails and execution tools.*

### Instructions: The "Engineering Constitution"
Instructions define what "good" looks like. These are the truths of our engineering culture that don't change from task to task—things like Bicep linting rules, AVM expectations, or how we handle PowerShell errors. These are always loaded and always respected by the agent.

### Skills: The Procedural Tools
Skills are for multi-step, repeatable work. For example, a git commit generator is a skill because it follows a set workflow: analyze the diff, then generate a message. On the other hand, being a Bicep expert is a knowledge domain that belongs in Instructions. Skills handle the heavy lifting like scaffolding or running Pester tests.

### Prompts: Workflow Invocations
Prompts are manual triggers for when you want to run a specific, high-value workflow. Unlike instructions, which enforce behavior, prompts are invocations. They act as the "Start" button for complex routines.

### Agents: Domain Personas
I try to avoid "agent swarms." Instead, I focus on one primary Platform Engineer agent supported by specialized personas. Specialists, like a Bicep or PowerShell agent, are only triggered for deep, domain-heavy reasoning when needed.

---

### Hierarchy of Control
In an agent-ready repo, the hierarchy is pretty clear: Instructions > Agents > Skills > Prompts. Instructions define how everything else behaves.

```text
.github/
  instructions/  # Culture, Expectations, Non-negotiables
  skills/        # Automated, procedural tools
  prompts/       # Reusable workflows
.agents.md       # The "System Prompt Extension" (Mandatory Context)
```

### Context Injection with `.agents.md`
The `.agents.md` file at the root acts as a persistent memory for any agent entering the workspace. It enforces repo-wide policies like mandatory retrieval (agents *must* read specific instruction files before touching code) and skill change contracts.

### Progressive Disclosure and "Expert Hooks"
One of the biggest challenges with AI agents is context fatigue. If you dump every rule and script into the agent's memory at once, it loses focus on the actual task.

I solved this using progressive disclosure in the `SKILL.md` files.

Instead of forcing the agent to remember everything, I use trigger keywords—what I call "Expert Hooks." For example, a Bicep skill description might say: *"Use this when the user mentions Azure, Bicep, or Infrastructure."*

By keeping the high-level descriptions concise, we protect the agent's token window. It only "opens" the full expertise—the complex regex for naming conventions or specific Bicep linter codes—when it identifies a task that actually requires it.

### Validation as a Coaching Loop
An agent is only as reliable as its feedback loop. In our repo, we've moved validation from "CI-only" to "In-Agent."

![Validation Coaching Loop](../../images/agent-ready-platform-repo/validation-loop.drawio.png)

*Figure 2: The self-healing loop where the agent triages compiler errors and auto-corrects based on remediation hints.*

We created a `bicep-validator` skill that doesn't just run a build and fail. It triages the output. If the agent makes a mistake, like a parameter mismatch, the tool provides a remediation hint directly in the agent's interaction flow.

This creates a "self-healing" experience. The agent breaks a rule, the tool coaches it on the fix, and the agent corrects itself—often before I even see the code. This turns validation from a gatekeeper into a tutor.

## Some technical details

### Choosing between AVM and custom modules
I taught the agents how to choose between Azure Verified Modules (AVM) and custom ones. The AVM skill includes a simple decision tree:
1. Always check for an existing AVM first.
2. If it exists but lacks a feature, document the gap.
3. Only build custom if there's no AVM available.

### PowerShell and Pester
For PowerShell automation, I integrated `PSScriptAnalyzer` and `Pester` into the workflow. The `powershell-module-scaffolder` skill ensures new functions use approved verbs like `Invoke` or `Get`, and it automatically generates a matching `.Tests.ps1` file.

## What I learned along the way

Building this was a shift in how I think about Infrastructure as Code.

*   **Constraints are actually helpful**: By giving the agent clear "never" and "mandatory" lists, it actually became more creative. It spends less time guessing what I want and more time solving the complex parts.
*   **Documentation is now code**: Your documentation is part of the execution path. If a standard isn't written in a way an agent can find it, it basically doesn't exist.
*   **Trust but verify**: I don't just trust the agent to read the docs. I use evaluation fixtures to prove that the skills actually work as intended.

## A quick checklist for "Agent-Readiness"

If you want to make your own repo agent-ready:
- [x] **Define your "Ground Truth"**: Use `.agents.md` for global rules.
- [x] **Modularize Instructions**: Keep your README from becoming a junk drawer.
- [x] **Build Validation Skills**: Give the agent tools to test its own work.
- [x] **Create Evaluation Fixtures**: Provide examples of "Good" and "Bad" code.

## Want to dive deeper?
Here are some of the resources I found useful while building this:
- [VS Code Copilot Documentation](https://code.visualstudio.com/docs/copilot/overview)
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Anthropic Skills Best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [agentskills.io](https://agentskills.io/home)
- [skills.sh](https://skills.sh/)

You can find the code I've been working on here: [platform-agent-kit](https://github.com/irwins/platform-agent-kit). It's still a work in progress, but it's a start.

By treating AI as a first-class citizen in the repo structure, the goal is to ensure that every line of code—whether written by me or an agent—meets the same standards. At least, that's what I'm aiming for! 😉

Ttyl,

Urv