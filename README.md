# Deep UX Cartographer — OpenClaw Deployment Package

Portable OpenClaw configuration for the two-phase UX Cartographer agent system.

## What's Inside

```
openclaw-deploy/
├── openclaw.json              # Main config (splits via $include)
├── .env.template              # Secrets template (copy to .env)
├── config/
│   ├── agents.json5           # Two agents: Phase 1 (surface) + Phase 2 (deep)
│   ├── channels.json5         # Control UI only (no external channels)
│   ├── gateway.json5          # Loopback bind, token auth
│   └── session.json5          # Per-channel-peer, daily reset
└── prompts/
    ├── phase1-system.md       # Phase 1: Surface mapping + recon
    └── phase2-system.md       # Phase 2: Deep execution + E2E output
```

## Architecture

Two separate agents with isolated workspaces:

| Agent | ID | Purpose | Prompt |
|-------|----|---------|--------|
| Phase 1 | `ux-cartographer-p1` | Surface mapping, feature discovery, gate detection | `phase1-system.md` |
| Phase 2 | `ux-cartographer-p2` | Deep journey execution, bug regression, E2E cases | `phase2-system.md` |

**Flow:** Orchestrator invokes Phase 1 → Phase 1 produces surface map → Orchestrator passes output to Phase 2 → Phase 2 appends deep execution results.

Both agents send **structured JSON heartbeats every 60 seconds** to the orchestrator.

## Setup on Target PC

### Prerequisites

- Node.js 22+ (`node --version`)
- npm or pnpm

### Step 1: Install OpenClaw

```bash
npm install -g openclaw
# or
pnpm add -g openclaw
```

### Step 2: Create config directory

```bash
mkdir -p ~/.openclaw/config
```

### Step 3: Copy deployment files

```bash
# From wherever you placed this package:
cp openclaw.json ~/.openclaw/
cp -r config/ ~/.openclaw/config/
cp -r prompts/ ~/.openclaw/prompts/
```

### Step 4: Configure secrets

```bash
cp .env.template ~/.openclaw/.env
```

Edit `~/.openclaw/.env` and fill in:

```
ANTHROPIC_API_KEY=sk-ant-YOUR_REAL_KEY
OPENCLAW_GATEWAY_TOKEN=any-strong-random-string
```

Generate a gateway token:
```bash
openssl rand -hex 32
```

### Step 5: Validate configuration

```bash
openclaw doctor
```

Fix any reported issues before proceeding.

### Step 6: Start the Gateway

```bash
openclaw gateway start
```

The Control UI will be available at `http://127.0.0.1:18789/`

## Usage

### Invoking Phase 1

Send a message to the Phase 1 agent with the target URL:

```
Map this platform: https://example.com/dashboard
```

Phase 1 will:
1. Run recon (Phase 0)
2. Present a plan and wait for approval
3. Execute surface mapping with heartbeats every 60s
4. Output a complete surface map + artifacts

### Invoking Phase 2

After Phase 1 completes, send its output to the Phase 2 agent:

```
@mapper-p2 Here is the Phase 1 output: [paste or attach Phase 1 artifact]
```

Phase 2 will:
1. Load and normalize Phase 1 output
2. Present an execution plan and wait for approval
3. Execute deep journeys with heartbeats every 60s
4. Append results to the Phase 1 artifact

### Heartbeat Protocol

Both agents emit structured JSON heartbeats every 60 seconds:

```json
{
  "heartbeat": true,
  "seq": 5,
  "status": "progressing",
  "phase": "phase1_journey",
  "coveragePct": 45.2,
  "nextAction": "Testing form validation on /settings"
}
```

A gap > 90 seconds without heartbeat means the agent has stalled.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Gateway won't start | Run `openclaw doctor` — strict schema validation catches all config errors |
| Agent not responding | Check `openclaw logs --follow` for errors |
| Wrong agent receives message | Check bindings in `config/agents.json5` — Phase 2 needs `@mapper-p2` mention |
| Missing env vars | Check `~/.openclaw/.env` — only UPPERCASE names work |

## File Permissions

After copying, verify:
```bash
chmod 600 ~/.openclaw/.env          # Secrets file — owner only
chmod 644 ~/.openclaw/openclaw.json # Config readable
```
