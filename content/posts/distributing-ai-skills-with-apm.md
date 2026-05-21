+++
date = '2026-05-21'
draft = false
title = 'Distributing AI Skills Across an Org with APM'
summary = "How a platform team can ship AI agent skills to every developer's laptop — versioned, updated automatically, and working across Claude Code and GitHub Copilot — using APM (Agent Package Manager)."
tags = ['ai', 'platform-engineering', 'devex', 'apm', 'claude-code', 'copilot']
categories = ['platform-engineering']

[cover]
image = "/images/posts/distributing-ai-skills-with-apm/cover.png"
alt = "Platform Engineering ships AI skills to every developer's laptop via APM"
+++

Without skills, every developer feeds the agent a raw prompt and prays. Sometimes it nails the team's conventions. Sometimes it invents some creative solution from StackOverflow that you now have to refactor by hand. The output quality variance is huge, and the only lever a developer has is "write a better prompt".

So the question for a platform team is not whether to write skills. It is: how do you ship them to every developer's laptop, keep them updated, and make it work across Claude Code AND GitHub Copilot, on macOS AND Windows?

This post is the short version of what worked.

![Where skills fit on the AI SDLC map](/images/posts/distributing-ai-skills-with-apm/skills-on-sdlc-map.png)

## Where skills fit on the AI SDLC map

Modern coding agents are often set up with a plan / execute split. One pass to think through what needs to happen, another to actually do it. Sometimes more roles are bolted on — a reviewer, an architect, a tester — but the load-bearing one is the executor.

When the spec lands on the executor, it knows WHAT to build. The problem is that it does not know HOW your team builds it. Naming conventions, where to place which resource, which shared component to integrate with, which MCP server to call for the policy document. That is the gap skills cover.

A skill is just a folder:

```
your-skill-name/
├── SKILL.md          # Required - description + instructions
├── scripts/          # Optional - executable code
├── references/       # Optional - static docs the agent reads on demand
└── assets/           # Optional - templates, etc.
```

The key mechanic here is **progressive disclosure**. Only the `description` field from `SKILL.md` is loaded into context at startup. The agent reads the full `SKILL.md` only when your prompt matches the description, and reads the `references/` files only when it actually needs them. You can ship a lot of context without burning a lot of tokens.

That also means the **description is the most important field you'll write**. It is the entire basis for discovery. Mine include the exact phrases developers actually type when they want this skill to fire — not paraphrases, the real prompts from real users.

## The distribution problem

So you have built the skills. Now what?

Each developer has a home directory with an AI tool config — `~/.claude/skills/` for Claude Code, `~/.copilot/skills/` for GitHub Copilot. In theory, you just get your skills into those folders on everyone's machine and you are done.

In practice this is where platform teams write:

- A README telling people which folders to copy.
- A shell script that clones a "golden repo" and rsyncs files.
- A Backstage template that scaffolds the files once and immediately starts drifting.

All three have the same failure mode: **no single source of truth, no versioning, no way to push a fix**. Copy-paste works right up until it does not, and then you are chasing down why the same skill behaves differently on five different laptops.

This is the exact problem `npm`, `pip`, and `cargo` solved for code dependencies decades ago. A manifest, a resolver, a lockfile, a reproducible install. Agent configuration deserves the same — and that is what APM gives you.

## Meet APM

[APM (Agent Package Manager)](https://github.com/microsoft/apm) is an open-source tool from Microsoft. It treats AI agent configuration the way npm treats JavaScript dependencies. You declare what you want in an `apm.yml`, run `apm install`, and APM resolves the tree, pins exact commits in a lockfile, and deploys files to the directories your AI tools actually read from — `.claude/`, `.github/`, `.cursor/`, `.opencode/`.

It understands several package shapes out of the box: full APM packages, Claude Skills, GitHub Copilot plugins, MCP servers, individual files from a monorepo. For a platform team, the two that matter most are **APM packages** (bundles of skills + instructions + agents + prompts) and **Claude Skills** (just a `SKILL.md` and its supporting files).

### What an APM package looks like

An APM package is a git repository with an `apm.yml` at the root and a `.apm/` directory holding the primitives:

```
backend-template/
├── apm.yml                  # Package manifest
├── README.md
└── .apm/
    ├── instructions/        # Coding standards (.instructions.md)
    ├── prompts/             # Slash commands (.prompt.md)
    ├── skills/              # Agent skills (SKILL.md folders)
    ├── agents/              # Personas (.agent.md)
    └── hooks/               # Event handlers (.json)
```

The `apm.yml` itself is minimal — a name, a version, and optionally its own dependencies and MCP servers:

```yaml
name: backend-template
version: 1.0.0
description: Standard skills for backend services
dependencies:
  apm: []
  mcp: []
```

You host the package on any git host APM can reach — GitHub, GitLab, Bitbucket, self-hosted. We host on GitLab.

### Skills and their MCP servers travel together

Here is a detail that matters a lot in practice: skills often need MCP servers to be useful. A "create Jira ticket" skill without the Atlassian MCP server is just a doc. A "check pipeline status" skill without the GitLab MCP server is nothing at all.

APM handles this. A package can declare its MCP dependencies right next to its APM dependencies:

```yaml
name: backend-template
version: 1.0.0
dependencies:
  apm: []
  mcp:
    - name: atlassian
      registry: false
      transport: http
      url: https://mcp.atlassian.com/v1/mcp
    - io.github.github/github-mcp-server
```

When a developer installs the package, APM walks the whole tree — including transitive packages — and collects every MCP server declared anywhere in it. The developer's agent gets both the skill AND the tools it needs to actually do something with that skill, from a single `apm install`.

For a platform team this is the piece that lets you ship skills as real, self-contained units. Not "here is a skill, please also go configure three MCP servers by hand first".

## The setup: user-level global skills

Most APM tutorials show you a **project-level** `apm.yml` committed to a repo. That is fine when skills are tied to one codebase. But a competence center's skills are not — they apply across dozens of services the developer works on. Asking every engineer to add an `apm.yml` to every repo is exactly the kind of ceremony that kills adoption.

The pattern that worked for us: **user-level, global skills**. One manifest per developer, in their home directory. Applies everywhere.

Three pieces:

**1. A package in GitLab that holds the team's "standard" skills** — for us this is the backend template skills produced by our Backstage template. A proper APM package with its own `apm.yml` and `.apm/skills/`.

**2. A monorepo of individual skills** — one-off skills that do not belong to any single template (company-specific integrations, knowledge skills, utilities). APM can pull individual subdirectories out of a monorepo, so each skill is independently installable.

**3. A user-level manifest at `~/.apm/apm.yml`** on each developer's machine that pulls both:

```yaml
name: my-workspace
version: 1.0.0
dependencies:
  apm:
    - gitlab.com/<yourorg>/apm-packages/backend-template
    - git: https://gitlab.com/<yourorg>/ai/skills.git
      path: repo-onboarding
    - git: https://gitlab.com/<yourorg>/ai/skills.git
      path: azure-secret-rotation
  mcp: []
```

Running `apm install -g` (the `-g` flag is what makes this global — it installs to `~/.apm/` instead of the current project) deploys these skills to `~/.claude/skills/` and `~/.copilot/skills/`. No plugins, no wrappers. The tools just pick them up.

## Keeping it fresh: the SessionStart hook

![Skill distribution flow](/images/posts/distributing-ai-skills-with-apm/skill-distribution-flow.png)

Shipping skills once is easy. Keeping them fresh across hundreds of machines is the real problem. Our answer: a `SessionStart` hook that runs `apm deps update` every time the agent starts.

For Claude Code, in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "apm deps update -g --target claude",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

For Copilot, the same idea lives in a `~/.copilot/hooks/` JSON file. Every time a developer starts a session, APM checks the lockfile, pulls any updated skills, and redeploys them. Five seconds of latency. Fresh skills forever.

This is the part that flips APM from "a package manager" to "a distribution system". The user never runs an update command. They never even think about it. Platform pushes a change to the skill monorepo. Next time the developer opens Claude Code, it is there.

## What skills look like in practice

A quick tour of the actual skill categories we ended up with. This is the part where I think other teams will recognise themselves.

**Static external knowledge.** A `platform-knowledge` skill teaches the agent how our internal infrastructure platform is laid out — where the shared ALB lives, how services join the cluster, how SSM contracts work. Pure reference material. The skill has `SKILL.md`, a `references/` folder with architecture markdown and diagrams, and `evals/evals.json` to verify the agent picks the skill up on the right prompts.

**Actual information lookup.** A CDK tags skill needs the company tag policy, which lives in Confluence and changes more often than I would like. The skill knows to call the Atlassian MCP server's `get_page` or `search_page` to fetch the current policy on demand. The skill itself is small. The Confluence page is the source of truth. No copy-paste, no drift.

**Company-specific systems.** Our GitLab is managed by a Terraform project — adding a new repo means writing the right entry in the right tfvars file, opening an MR, asking the right reviewer questions. A `repo-onboarding` skill walks the agent through all of it.

**Integrations.** An `auth-integration` skill wires up ABAC authorization for a .NET microservice — NuGet packages, the PEP handler, DI registration, the policy manifest. This kind of multi-step, multi-file integration is exactly the place where a raw prompt produces "creative" code that nobody wants to review. With the skill, the result is the same shape every time.

The pattern: each skill captures something a platform team would otherwise be writing into a wiki page that nobody reads.

## Onboarding: one prompt

The bootstrap still has to happen somewhere — install APM, write the manifest, add MCP servers, set up the hook. But you do not have to teach it. You ship a setup prompt that the developer pastes into Claude Code on a fresh machine, and the agent does it all:

> You are setting up the APM (Agent Package Manager) skill distribution system on my machine. Install APM, configure MCP servers, write `~/.apm/apm.yml`, add the `SessionStart` hook, and run the initial update.

The real prompt sets a `TARGET_AGENT` and a `TARGET_OS` variable at the top, then walks through five concrete steps with the exact commands for each combination. The agent installs itself, basically. New hires go from zero to fully configured in a few minutes with no platform-team babysitting.

## What I would watch out for

A few things worth knowing before you copy this pattern.

- **Target detection needs the directory to exist.** `apm install` only deploys to `~/.claude/` or `~/.copilot/` if those directories are already there. Run the agent once before the first `apm install`, or create the directories manually in the setup prompt.
- **Lock down the MCP permissions.** When you add the Atlassian or GitLab MCP servers, the default tool surface is wide. Explicitly deny the write-side tools in `settings.json` (`createConfluencePage`, `editJiraIssue`, etc.) unless you really want the agent editing tickets on its own.
- **Skill descriptions are your API.** If discovery breaks, the skill might as well not exist. Test with the real phrases developers actually use, not the ones you wish they used.

## The shift

The thing that surprised me most building this: it is not really about APM. APM is fine, it works, whatever. The real shift is treating AI skills as a **platform product**.

Competence center writes them once. Distribution, versioning, and updates are automated. Developers do not copy files or follow READMEs. They open their agent and it knows how the company builds things.

That is the same operating model we have had for code dependencies for twenty years. It was only a matter of time before agent context got it too.
