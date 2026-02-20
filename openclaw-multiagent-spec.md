# OpenClaw Multi-Agent Team — Complete Setup Specification

> **Hardware:** Mac Mini M1 · **Agents:** 4 · **Integration:** Slack + GitHub · **Model:** Feb 2026

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Hardware & OS](#2-hardware--os)
3. [OpenClaw Installation](#3-openclaw-installation)
4. [Agent Definitions](#4-agent-definitions)
5. [OpenClaw Configuration](#5-openclaw-configuration)
6. [Slack Integration](#6-slack-integration)
7. [GitHub Integration](#7-github-integration)
8. [Security Configuration](#8-security-configuration)
9. [Agent Coordination](#9-agent-coordination)
10. [API Cost Management](#10-api-cost-management)
11. [Maintenance & Monitoring](#11-maintenance--monitoring)

---

## 1. System Overview

Four agents run on a dedicated Mac Mini M1 as isolated macOS users with full native shell execution, Docker access via a scoped socket proxy, GitHub integration, and Slack communication. Security is enforced through five independent layers with no single point of failure.

### Design Principles

- **Dedicated hardware** — Mac Mini used exclusively for the agent team, never for daily work
- **Native execution** — agents run directly on the host OS for full coding capability, not wrapped in Docker
- **OS-level isolation** — each agent is a separate macOS user; cross-agent file access is physically impossible
- **Defense in depth** — five independent security layers, any one of which limits blast radius
- **Capability-first** — security constrains the blast radius, not the legitimate workflow

### Architecture

```
Mac Mini M1  (dedicated, always-on)
│
├── macOS User: oc-gateway  ──→  OpenClaw Gateway Process
│                                 (routes messages, no tool execution)
│
├── macOS User: oc-claw     ──→  Agent: Claw      (Sysadmin)
├── macOS User: oc-bernard  ──→  Agent: Bernard   (Developer)
├── macOS User: oc-vale     ──→  Agent: Vale      (Marketer)
├── macOS User: oc-gumbo    ──→  Agent: Gumbo     (PM / Glue)
│
├── Shared Group: oc-agents ──→  /shared-brain/  (read-write for all agents)
│
├── Docker Socket Proxy     ──→  tcp://127.0.0.1:2375
│   (scoped Docker access)         no privileged, no host mounts
│
├── Squid Proxy             ──→  localhost:3128
│   (all outbound HTTP)            domain allowlist only
│
└── OpenClaw Gateway        ──→  127.0.0.1:18789  (loopback only)
    (Tailscale for remote           + gateway auth token
     access from your main machine)
```

### Security Layer Summary

| Layer | Mechanism | Enforced By | Protects Against |
|-------|-----------|-------------|-----------------|
| 1 | OS User Isolation | macOS kernel | Cross-agent file access, credential theft |
| 2 | macOS Sandbox Profile | macOS kernel (sandboxd) | Filesystem escape, arbitrary network calls |
| 3 | Docker Socket Proxy | tecnativa/docker-socket-proxy | Privileged containers, host mounts, infra access |
| 4 | Exec Approvals + Allowlist | OpenClaw exec policy | Prompt injection → shell execution |
| 5 | AGENTS.md Policy | Claude Opus (model reasoning) | Behavioral guardrails, task scope |

---

## 2. Hardware & OS

### 2.1 Mac Mini M1 Specification

| Spec | Detail |
|------|--------|
| RAM | 16GB Unified Memory minimum — 8GB insufficient for 4 concurrent agents |
| Storage | 512GB SSD minimum |
| OS | macOS Ventura 13.x or later |
| Network | Wired Ethernet preferred — agents run 24/7 |
| Remote Access | Screen Sharing + SSH enabled, Tailscale installed |
| Role | Dedicated solely to OpenClaw — no personal use |

> **M1 Capacity Note:** CPU is idle >90% of the time — agent tasks are API-latency-bound, not compute-bound. All 4 agents + Docker containers + Gateway ≈ 6–8GB typical memory use. Your real ceiling will be Anthropic API rate limits, not hardware.

### 2.2 macOS User Accounts

Create five standard (non-admin) macOS user accounts:

| Username | Role | Admin? |
|----------|------|--------|
| `oc-gateway` | OpenClaw Gateway orchestrator | No |
| `oc-claw` | Agent: Claw (Sysadmin) | No |
| `oc-bernard` | Agent: Bernard (Developer) | No |
| `oc-vale` | Agent: Vale (Marketer) | No |
| `oc-gumbo` | Agent: Gumbo (Project Manager) | No |

```bash
# Create each user (run as admin)
for user in oc-gateway oc-claw oc-bernard oc-vale oc-gumbo; do
  sudo dscl . -create /Users/$user
  sudo dscl . -create /Users/$user UserShell /bin/zsh
  sudo dscl . -create /Users/$user RealName "OpenClaw $user"
  sudo dscl . -create /Users/$user NFSHomeDirectory /Users/$user
  sudo createhomedir -c -u $user
done

# Create shared group for brain folder access
sudo dscl . -create /Groups/oc-agents
sudo dscl . -create /Groups/oc-agents GroupMembership oc-claw oc-bernard oc-vale oc-gumbo

# Create shared brain folder
sudo mkdir -p /shared-brain
sudo chown root:oc-agents /shared-brain
sudo chmod 2775 /shared-brain   # setgid: new files inherit oc-agents group
```

---

## 3. OpenClaw Installation

### 3.1 Prerequisites

- Node.js 22.12.0 LTS or later
- pnpm: `npm install -g pnpm`
- Docker Desktop for Mac — running
- Tailscale — installed and joined to your tailnet
- Git + GitHub CLI (`gh`) — installed via Homebrew

### 3.2 Install (as oc-gateway user)

```bash
# Switch to gateway user
sudo -u oc-gateway -i

# Install OpenClaw
npm install -g openclaw

# Verify
openclaw --version

# Run initial setup wizard
openclaw onboard

# Immediately run security audit after onboard
openclaw security audit --deep --fix
```

### 3.3 Directory Structure

```
/Users/oc-gateway/.openclaw/
├── openclaw.json              ← master config (agents, channels, bindings)
├── .env                       ← all secrets (chmod 600)
├── exec-approvals.json        ← exec policy per agent
├── workspace-claw/            ← Claw agent workspace
├── workspace-bernard/         ← Bernard agent workspace
├── workspace-vale/            ← Vale agent workspace
├── workspace-gumbo/           ← Gumbo agent workspace
├── agents/
│   ├── claw/agent/            ← auth profiles + sessions
│   ├── bernard/agent/
│   ├── vale/agent/
│   └── gumbo/agent/
└── skills/                    ← curated skills only, no ClawHub auto-install

/shared-brain/                 ← shared across all agents (group: oc-agents)
├── projects.md
├── context.md
├── tasks.md
└── decisions.md
```

---

## 4. Agent Definitions

All four agents have identical capability profiles — full native shell, Docker, GitHub, web access. Differentiation is behavioral (SOUL.md + AGENTS.md) and model choice, not permission restriction.

### Agent Roster

| Agent | Role | Model | OS User | Slack Bot |
|-------|------|-------|---------|-----------|
| **Claw** | System Administrator | `claude-opus-4-5` | `oc-claw` | `@claw-bot` |
| **Bernard** | Developer | `claude-opus-4-5` | `oc-bernard` | `@bernard-bot` |
| **Vale** | Marketer | `claude-sonnet-4-20250514` | `oc-vale` | `@vale-bot` |
| **Gumbo** | Project Manager / Glue | `claude-sonnet-4-20250514` | `oc-gumbo` | `@gumbo-bot` |

> **Model assignment rationale:** Claw and Bernard use Opus because they have shell execution access — Opus has the strongest prompt injection resistance. Vale and Gumbo use Sonnet because they do primarily read, web search, and coordination tasks — significantly cheaper and faster for high-frequency use.

### 4.1 SOUL.md — Identity & Behavioral Guardrails

Place in each agent's workspace root. Example for Bernard:

```markdown
# Bernard — Developer Agent

## Identity
You are Bernard, a senior software developer. You are precise, methodical,
and never cut corners on correctness. You prefer to ask for clarification
over making assumptions.

## Hard Rules
- Never run commands outside your approved exec allowlist without explicit Slack approval
- Never read or write files outside your workspace directory
- Never push directly to main or master branches — always use feature/* branches
- Never use --privileged or --network=host with Docker
- Never mount host paths into Docker containers
- If uncertain about scope or authorization, stop and ask in Slack before proceeding

## Docker Policy
- Only manage containers with the prefix "bernard-"
- Use DOCKER_HOST=tcp://127.0.0.1:2375 (socket proxy) always
- Approved base images: node:*, python:*, alpine:*, ubuntu:22.04, postgres:*, redis:*
```

### 4.2 AGENTS.md — Execution Policy

```markdown
# Execution Security Policy

1. SANDBOX-FIRST: Default to least-privilege. Expand only when required.
2. ALLOWLIST: Only run binaries in the approved exec allowlist.
3. WORKSPACE-ONLY: All file reads/writes stay within your workspace directory.
4. APPROVAL-ON-MISS: Any command not in allowlist → send Slack approval request.
5. WRITE-SCOPE: Never write to paths containing: .ssh, .aws, .config, Library, /etc.

## High-Risk Operations (Always Ask)
- rm, rmdir, mv  (destructive)
- curl, wget     (arbitrary outbound)
- chmod, chown   (permission changes)
- sudo           (should never be needed)
- git push       (verify branch before executing)

## Approved Workflow
- git clone/pull/status/diff/add/commit/push (feature/* branches only)
- npm install/run/test/build
- node, npx, nvm
- docker (via socket proxy at tcp://127.0.0.1:2375)
- gh (GitHub CLI — issues, PRs, comments)
- ls, cat, grep, find, diff (read-only inspection)
```

---

## 5. OpenClaw Configuration

### 5.1 Master Config (openclaw.json)

```json
{
  "gateway": {
    "bind": "127.0.0.1",
    "port": 18789,
    "auth": {
      "password": "${OPENCLAW_GATEWAY_PASSWORD}"
    },
    "controlUi": {
      "allowInsecureAuth": false
    }
  },

  "agents": {
    "list": [
      {
        "id": "claw",
        "workspace": "~/.openclaw/workspace-claw",
        "agentDir": "~/.openclaw/agents/claw/agent",
        "model": { "id": "anthropic/claude-opus-4-5" }
      },
      {
        "id": "bernard",
        "workspace": "~/.openclaw/workspace-bernard",
        "agentDir": "~/.openclaw/agents/bernard/agent",
        "model": { "id": "anthropic/claude-opus-4-5" }
      },
      {
        "id": "vale",
        "workspace": "~/.openclaw/workspace-vale",
        "agentDir": "~/.openclaw/agents/vale/agent",
        "model": { "id": "anthropic/claude-sonnet-4-20250514" }
      },
      {
        "id": "gumbo",
        "workspace": "~/.openclaw/workspace-gumbo",
        "agentDir": "~/.openclaw/agents/gumbo/agent",
        "model": { "id": "anthropic/claude-sonnet-4-20250514" }
      }
    ],
    "defaults": {
      "sandbox": { "mode": "off" },
      "tools": {
        "exec": {
          "host": "gateway",
          "applyPatch": { "workspaceOnly": true }
        },
        "fs": { "workspaceOnly": true }
      }
    }
  },

  "channels": {
    "slack": {
      "dmPolicy": "pairing",
      "groupPolicy": "allowlist",
      "allowFrom": ["YOUR_SLACK_USER_ID"],
      "accounts": {
        "claw-bot":    { "token": "${SLACK_BOT_TOKEN_CLAW}" },
        "bernard-bot": { "token": "${SLACK_BOT_TOKEN_BERNARD}" },
        "vale-bot":    { "token": "${SLACK_BOT_TOKEN_VALE}" },
        "gumbo-bot":   { "token": "${SLACK_BOT_TOKEN_GUMBO}" }
      }
    }
  },

  "bindings": [
    { "agentId": "claw",    "match": { "channel": "slack", "accountId": "claw-bot" } },
    { "agentId": "bernard", "match": { "channel": "slack", "accountId": "bernard-bot" } },
    { "agentId": "vale",    "match": { "channel": "slack", "accountId": "vale-bot" } },
    { "agentId": "gumbo",   "match": { "channel": "slack", "accountId": "gumbo-bot" } }
  ],

  "plugins": {
    "autoAllowSkills": false,
    "allow": [
      "openclaw/skill-github",
      "openclaw/skill-slack",
      "openclaw/skill-calendar"
    ]
  },

  "logging": {
    "level": "info",
    "redactSensitive": "tools",
    "sessions": true,
    "toolCalls": true
  }
}
```

### 5.2 Environment Variables (.env)

```bash
# chmod 600 this file immediately after creation

# API — route all calls via OpenRouter
OPENROUTER_API_KEY=sk-or-...
OPENCLAW_GATEWAY_PASSWORD=<strong-random-password>

# Slack bot tokens (one per agent)
SLACK_BOT_TOKEN_CLAW=xoxb-...
SLACK_BOT_TOKEN_BERNARD=xoxb-...
SLACK_BOT_TOKEN_VALE=xoxb-...
SLACK_BOT_TOKEN_GUMBO=xoxb-...

# GitHub PATs (one per agent, scoped to their repos)
GITHUB_TOKEN_CLAW=ghp_...
GITHUB_TOKEN_BERNARD=ghp_...
GITHUB_TOKEN_VALE=ghp_...
GITHUB_TOKEN_GUMBO=ghp_...
```

```bash
chmod 600 ~/.openclaw/.env
chmod 700 ~/.openclaw/
```

### 5.3 Exec Approvals (exec-approvals.json)

Every agent gets the same full development tooling. Commands not in the allowlist trigger a Slack approval request before execution.

```json
{
  "version": 1,
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny"
  },
  "agents": {
    "*": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "allowlist": [
        "/opt/homebrew/bin/git",    "/usr/local/bin/git",
        "/opt/homebrew/bin/node",   "/usr/local/bin/node",
        "/opt/homebrew/bin/npm",    "/usr/local/bin/npm",
        "/opt/homebrew/bin/npx",    "/usr/local/bin/npx",
        "/opt/homebrew/bin/pnpm",
        "/opt/homebrew/bin/python3", "/usr/bin/python3",
        "/opt/homebrew/bin/pip3",
        "/usr/local/bin/docker",    "/opt/homebrew/bin/docker",
        "/opt/homebrew/bin/gh",     "/usr/local/bin/gh",
        "/usr/bin/ls",  "/usr/bin/cat",  "/usr/bin/grep",
        "/usr/bin/find", "/usr/bin/diff", "/usr/bin/wc",
        "/usr/bin/head", "/usr/bin/tail", "/usr/bin/sort"
      ]
    }
  }
}
```

---

## 6. Slack Integration

### 6.1 Create Four Slack Bot Apps

Repeat the following for each agent (Claw, Bernard, Vale, Gumbo):

1. Go to **api.slack.com/apps** → Create New App → From Scratch
2. Name it to match the agent, e.g. `"Bernard - Dev Agent"`. Select your workspace.
3. Under **OAuth & Permissions**, add Bot Token Scopes:
   - `chat:write`
   - `im:history` + `im:read`
   - `channels:read` + `channels:history`
   - `app_mentions:read`
   - `reactions:write`
4. **Install to Workspace**. Copy the `xoxb-` Bot Token into `.env`
5. Under **Event Subscriptions**, subscribe to: `message.im` and `app_mention`
6. Point the Request URL at your OpenClaw gateway webhook via Tailscale

> **Note:** If you add OAuth scopes after the initial install, new scopes are not active until you reinstall the app. Missing scopes are the leading cause of silent bots that appear connected but never respond.

### 6.2 Slack Channel Structure

| Channel | Purpose |
|---------|---------|
| `#claw` | Sysadmin tasks — direct to Claw |
| `#bernard` | Development tasks — direct to Bernard |
| `#vale` | Marketing tasks — direct to Vale |
| `#gumbo` | Project management, task dispatch |
| `#ai-team` | All four bots present — cross-agent coordination |
| `#ai-approvals` | Exec approval requests — all agents post here for human sign-off |
| `#ai-logs` | Daily activity summaries and API cost reports |

---

## 7. GitHub Integration

### 7.1 Personal Access Tokens

Create one fine-grained GitHub PAT per agent:

| Agent | Scopes | Repo Access |
|-------|--------|-------------|
| Claw | `repo` (full), `admin:org` (read), `workflow` | All repos |
| Bernard | `repo` (full), `issues`, `pull_requests`, `workflows` | Assigned code repos only |
| Vale | `repo:contents` (read), `issues` (read) | Docs/content repos only |
| Gumbo | `issues` (read/write), `projects` | All repos (issues only) |

### 7.2 Authenticate gh CLI Per Agent

```bash
# Run as each agent user
sudo -u oc-bernard -i

gh auth login --with-token <<< "${GITHUB_TOKEN_BERNARD}"
gh auth status

# Verify issue access
gh issue list --repo yourorg/yourrepo --limit 5
gh issue view 42
gh issue comment 42 --body "Picked up — working on a fix"
gh pr create --title "Fix: issue #42" --body "..." --base main --head feature/fix-42
```

### 7.3 Git Identity Per Agent

```bash
sudo -u oc-bernard -i

git config --global user.name "Bernard (OpenClaw Dev Agent)"
git config --global user.email "bernard-agent@yourdomain.com"
git config --global credential.helper store

echo "https://x-token:${GITHUB_TOKEN_BERNARD}@github.com" > ~/.git-credentials
chmod 600 ~/.git-credentials
```

---

## 8. Security Configuration

### 8.1 macOS Sandbox Profiles

Kernel-enforced — prompt injection cannot bypass this because enforcement happens in the OS, not in Node.js.

```scheme
; /Users/oc-gateway/.openclaw/sandbox-agents.sb
; Launch with: sandbox-exec -f sandbox-agents.sb openclaw gateway run

(version 1)
(deny default)

; Network: HTTPS outbound + loopback only
(allow network-outbound
  (remote ip "0.0.0.0:443")
  (remote ip "127.0.0.1:18789")   ; OpenClaw gateway
  (remote ip "127.0.0.1:3128")    ; Squid proxy
  (remote ip "127.0.0.1:2375"))   ; Docker socket proxy

; Filesystem reads: workspace, shared brain, approved tools
(allow file-read*
  (subpath (string-append (getenv "HOME") "/.openclaw"))
  (subpath "/shared-brain")
  (subpath "/opt/homebrew/bin")
  (subpath "/usr/local/bin")
  (subpath "/usr/bin")
  (subpath "/usr/lib"))

; Filesystem writes: agent workspace and temp only
(allow file-write*
  (subpath (string-append (getenv "HOME") "/.openclaw/workspace-" (getenv "AGENT_ID")))
  (subpath "/tmp/openclaw"))

; Explicitly deny sensitive paths
(deny file-read*
  (subpath (string-append (getenv "HOME") "/.ssh"))
  (subpath (string-append (getenv "HOME") "/Library/Keychains"))
  (subpath (string-append (getenv "HOME") "/Library/Passwords")))

; Process execution: approved binaries only
(allow process-exec
  (literal "/opt/homebrew/bin/node")
  (literal "/opt/homebrew/bin/git")
  (literal "/opt/homebrew/bin/gh")
  (literal "/usr/local/bin/docker"))
```

### 8.2 Docker Socket Proxy

Agents get full development Docker workflows (start/stop containers, build images, exec in) while infrastructure-level operations are blocked.

```yaml
# docker-compose.yml
version: "3.9"
services:
  docker-proxy:
    image: tecnativa/docker-socket-proxy
    restart: always
    environment:
      CONTAINERS: 1   # list, inspect, start, stop
      START: 1
      STOP: 1
      RESTART: 1
      EXEC: 1         # exec into containers
      BUILD: 1        # build images
      IMAGES: 1       # list, pull images
      INFO: 1         # needed by docker CLI
      # BLOCKED:
      NETWORKS: 0     # cannot create/modify networks
      VOLUMES: 0      # cannot create volumes
      SWARM: 0
      SERVICES: 0
      SECRETS: 0
      CONFIGS: 0
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "127.0.0.1:2375:2375"
```

Add to each agent's shell profile:
```bash
export DOCKER_HOST=tcp://127.0.0.1:2375
```

### 8.3 Squid Egress Proxy

All outbound HTTP/HTTPS routes through Squid. Stops data exfiltration even if prompt injection succeeds.

```
# /etc/squid/openclaw-allowlist.txt
.slack.com
.anthropic.com
openrouter.ai
.github.com
api.github.com
.npmjs.org
.npmjs.com
registry.npmjs.org
pypi.org
files.pythonhosted.org
```

```
# squid.conf (minimal)
http_port 3128
acl allowed dstdomain "/etc/squid/openclaw-allowlist.txt"
http_access allow localhost allowed
http_access deny all
access_log /var/log/squid/access.log combined
```

### 8.4 Skills Policy

```json
{
  "plugins": {
    "autoAllowSkills": false,
    "allow": [
      "openclaw/skill-github",
      "openclaw/skill-slack",
      "openclaw/skill-calendar"
    ]
  }
}
```

> **ClawHub is blocked.** As of February 2026, ClawHub had 341 confirmed malicious skills (12–20% of the registry). `autoAllowSkills: false` blocks all automatic installation. Preferred approach: write custom skills as Markdown files in each agent's `workspace/skills/` folder. Zero supply chain risk, full visibility.

---

## 9. Agent Coordination

### 9.1 Communication Mechanisms

| Mechanism | How It Works |
|-----------|-------------|
| **Shared Brain Folder** | `/shared-brain/` mounted for all agents. One agent writes task completion or project updates; others read on next turn. |
| **sessions_send** | OpenClaw built-in. Agent A sends a message directly to Agent B's session. Used for task handoffs and data passing. |
| **sessions_spawn** | An agent spawns a sub-agent session for parallel sub-tasks. Primary cost tool — spin up cheaper model for simple subtasks. |
| **Slack Threading** | Each agent is a separate Slackbot. Threaded replies in `#ai-team` make coordination visible and auditable. |
| **Gumbo as PM** | Gumbo's role is coordination and glue. Dispatch tasks, track status, maintain queue in `/shared-brain/tasks.md`. |

### 9.2 Typical Workflow: Fix a GitHub Issue

1. You send to `#gumbo`: _"Issue #42 is blocking the sprint — get Bernard to fix it"_
2. Gumbo reads the issue via `gh` CLI, updates `/shared-brain/tasks.md`, sends full context to Bernard via `sessions_send`
3. Bernard pulls the repo, writes a fix, runs `npm test`
4. Bernard hits an unfamiliar command → exec approval request appears in `#ai-approvals` → you approve
5. Bernard pushes to `feature/fix-42`, creates a PR, comments on the original issue, reports back to Gumbo
6. Gumbo marks the task complete and posts a summary to `#ai-team`

### 9.3 Task Queue Format

```markdown
# /shared-brain/tasks.md

## Queue
| ID  | Agent   | Task                     | Status   | Created    |
|-----|---------|--------------------------|----------|------------|
| 001 | bernard | Fix issue #42            | complete | 2026-02-20 |
| 002 | vale    | Draft Q1 newsletter      | active   | 2026-02-20 |
| 003 | claw    | Review disk usage report | pending  | 2026-02-20 |
```

---

## 10. API Cost Management

### 10.1 OpenRouter Setup

Route all agent API calls through OpenRouter for unified cost tracking and rate limit management.

| Setting | Value |
|---------|-------|
| API Endpoint | `https://openrouter.ai/api/v1` |
| Claw / Bernard model | `anthropic/claude-opus-4-5` |
| Vale / Gumbo model | `anthropic/claude-sonnet-4-20250514` |
| Fallback chain | Opus → Sonnet → minimax/MiniMax-M2.1 |
| Spending alert | $20/day — configure in OpenRouter dashboard |
| Hard cap | $50/day — prevents runaway loops |

### 10.2 Cost Estimates

| Scenario | Daily Cost (est.) |
|----------|------------------|
| Claw + Bernard (Opus), moderate coding | ~$15/agent |
| Vale + Gumbo (Sonnet), research + coordination | ~$3/agent |
| Idle agents, scheduled tasks only | <$1/agent |
| First 2 days while calibrating | Budget $50–100 total |

> Casel's $200 in 2 days was with no spending caps and all agents on Opus.

---

## 11. Maintenance & Monitoring

### 11.1 Automated Health Checks

```bash
# crontab for oc-gateway user

# Security audit every Sunday midnight
0 0 * * 0 openclaw security audit --deep --fix >> /var/log/openclaw/audit.log 2>&1

# Doctor check daily at 6am
0 6 * * * openclaw doctor --json >> /var/log/openclaw/doctor.log 2>&1

# Weekly update check
0 1 * * 1 npm outdated -g openclaw >> /var/log/openclaw/updates.log 2>&1
```

### 11.2 After Every OpenClaw Update

1. Read the CHANGELOG and GitHub Security Advisories before updating
2. Run `openclaw security audit --deep --fix` immediately after
3. Check `exec-approvals.json` — updates can reset approval policies
4. Verify Slack bots still respond — updates occasionally require token reauth
5. Run `openclaw doctor` and resolve any new warnings before resuming

### 11.3 Files to Back Up

| Path | Contents |
|------|----------|
| `~/.openclaw/openclaw.json` | Master config |
| `~/.openclaw/.env` | All secrets — store separately, never in git |
| `~/.openclaw/exec-approvals.json` | Exec policy |
| `~/.openclaw/workspace-*/` | All agent workspaces (SOUL.md, AGENTS.md, memory) |
| `/shared-brain/` | Shared context and task history |

---

## What This Setup Gives You

- Four persistent AI agents available 24/7 via Slack on a Mac Mini you already own
- Full native shell execution — npm builds, Docker, GitHub, test suites all work natively
- GitHub Issues integration — agents read, comment, and create PRs exactly like a developer would
- Five independent security layers — blast radius bounded even if an agent is compromised
- Supply chain protection — ClawHub blocked, only curated skills allowed
- Human-in-the-loop — unknown commands hit Slack for approval before executing
- Cost control — OpenRouter with daily caps, Sonnet for routine tasks

> **Important:** Do not connect these agents to corporate VPNs, internal systems, or any infrastructure touching firm data. This is a personal productivity setup only.
