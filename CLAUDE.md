# NanoClaw

Personal Claude assistant. See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

Single Node.js process with skill-based channel system. Channels (WhatsApp, Telegram, Slack, Discord, Gmail) are skills that self-register at startup. Messages route to Claude Agent SDK running in containers (Linux VMs). Each group has isolated filesystem and memory.

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel registry (self-registration at startup) |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations |
| `groups/{name}/CLAUDE.md` | Per-group memory (isolated) |
| `container/skills/` | Skills loaded inside agent containers (browser, status, formatting) |

## Secrets / Credentials / Proxy (OneCLI)

API keys, secret keys, OAuth tokens, and auth credentials are managed by the OneCLI gateway — which handles secret injection into containers at request time, so no keys or tokens are ever passed to containers directly. Run `onecli --help`.

## Skills

Four types of skills exist in NanoClaw. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full taxonomy and guidelines.

- **Feature skills** — merge a `skill/*` branch to add capabilities (e.g. `/add-telegram`, `/add-slack`)
- **Utility skills** — ship code files alongside SKILL.md (e.g. `/claw`)
- **Operational skills** — instruction-only workflows, always on `main` (e.g. `/setup`, `/debug`)
- **Container skills** — loaded inside agent containers at runtime (`container/skills/`)

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |
| `/update-nanoclaw` | Bring upstream NanoClaw updates into a customized install |
| `/init-onecli` | Install OneCLI Agent Vault and migrate `.env` credentials to it |
| `/qodo-pr-resolver` | Fetch and fix Qodo PR review issues interactively or in batch |
| `/get-qodo-rules` | Load org- and repo-level coding rules from Qodo before code tasks |

## Contributing

Before creating a PR, adding a skill, or preparing any contribution, you MUST read [CONTRIBUTING.md](CONTRIBUTING.md). It covers accepted change types, the four skill types and their guidelines, SKILL.md format rules, PR requirements, and the pre-submission checklist (searching for existing PRs/issues, testing, description format).

## Development

Run commands directly—don't tell the user to run them.

```bash
npm run dev          # Run with hot reload
npm run build        # Compile TypeScript
```

## Container Image

The agent container image is built and published via GitHub Actions, not locally. **Do not run `./container/build.sh` on the remote host** — pull the versioned image from the registry instead.

**Registry:** `ghcr.io/kmjungersen/nanoclaw-agent`
**Workflow:** `.github/workflows/container-build.yml`
**Triggers:** `v*` tags (build + push), PRs touching `container/**` (build-only)

### Publishing a new image

```bash
# Tag and push — GH Actions builds and publishes automatically
git tag v1.2.48
git push origin v1.2.48
```

Tags produced: `1.2.48`, `1.2`, `1`, `latest` (multi-arch: amd64 + arm64).

### Deploying to the remote host

```bash
# Pull the latest published image
ssh nanoclaw-docker "docker pull ghcr.io/kmjungersen/nanoclaw-agent:latest"

# Retag so NanoClaw's orchestrator finds it (expects nanoclaw-agent:latest)
ssh nanoclaw-docker "docker tag ghcr.io/kmjungersen/nanoclaw-agent:latest nanoclaw-agent:latest"

# Clear stale agent-runner caches and restart
ssh nanoclaw-docker "cd ~/nanoclaw && rm -rf data/sessions/*/agent-runner-src && systemctl --user restart nanoclaw"
```

### Local builds (development only)

```bash
cd container && ./build.sh                          # local default
IMAGE_REGISTRY=ghcr.io/kmjungersen ./build.sh      # with registry prefix
```

Service management:
```bash
# macOS (launchd)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # restart

# Linux (systemd)
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
```

## Remote Deployment

NanoClaw runs on a remote EC2 instance, not locally. All operations must go through SSH.

**Instance:** `i-0a18ae97ecc4747b5` (t3.large, us-east-2, private IP `10.2.23.72`)
**SSH config alias:** `nanoclaw-docker` (configured in `~/.ssh/config`)
**SSH key:** `/Users/kmjungersen/Library/CloudStorage/GoogleDrive-kurtis@steelridge.io/Shared drives/SRP EXECUTIVE MANAGEMENT/IT SYSTEMS/AWS SERVER KEYS/SRP-DOCKER-1.pem`
**Local .env backup:** `.env.remote-backup` (gitignored)

### SSH Access

```bash
# Quick access via config alias
ssh nanoclaw-docker "command here"

# Or full path
ssh -i "$SSH_KEY" ec2-user@10.2.23.72 "command here"
```

Always use `-o ConnectTimeout=10` if SSH might hang. Never run SSH commands in background (`run_in_background`) — this exhausts connection limits and causes the instance to become unreachable.

### Service Management (on remote host)

```bash
# Status
ssh nanoclaw-docker "systemctl --user status nanoclaw | head -10"

# Restart
ssh nanoclaw-docker "systemctl --user restart nanoclaw"

# Logs
ssh nanoclaw-docker "cd ~/nanoclaw && tail -30 logs/nanoclaw.log"

# Container logs (find container name from nanoclaw.log first)
ssh nanoclaw-docker "docker logs --tail 30 nanoclaw-telegram-main-TIMESTAMP 2>&1"
```

### Building & Deploying Changes

```bash
# Build TypeScript on remote (for orchestrator changes)
ssh nanoclaw-docker "cd ~/nanoclaw && npm install && npm run build && systemctl --user restart nanoclaw"

# Deploy new agent container (after publishing via GH Actions)
ssh nanoclaw-docker "docker pull ghcr.io/kmjungersen/nanoclaw-agent:latest && docker tag ghcr.io/kmjungersen/nanoclaw-agent:latest nanoclaw-agent:latest && cd ~/nanoclaw && rm -rf data/sessions/*/agent-runner-src && systemctl --user restart nanoclaw"
```

### OneCLI (Credential Gateway)

```bash
# Health check
ssh nanoclaw-docker "curl -s http://127.0.0.1:10254/api/health"

# List secrets
ssh nanoclaw-docker "export PATH=\$HOME/.local/bin:\$PATH && onecli secrets list"

# OneCLI compose (runs on remote Docker)
ssh nanoclaw-docker "docker compose -p onecli -f ~/.onecli/docker-compose.yml ps"

# Restart OneCLI
ssh nanoclaw-docker "docker compose -p onecli -f ~/.onecli/docker-compose.yml restart"
```

Note: OneCLI's `docker-compose.yml` at `~/.onecli/` was modified to bind to `0.0.0.0` (not `127.0.0.1`) so agent containers can reach the gateway via `host.docker.internal`.

### Database Queries

```bash
# List registered groups
ssh nanoclaw-docker "cd ~/nanoclaw && npx tsx -e \"
import Database from 'better-sqlite3';
const db = new Database('store/messages.db');
console.log(JSON.stringify(db.prepare('SELECT * FROM registered_groups').all(), null, 2));
\""

# Check recent messages
ssh nanoclaw-docker "cd ~/nanoclaw && npx tsx -e \"
import Database from 'better-sqlite3';
const db = new Database('store/messages.db');
console.log(JSON.stringify(db.prepare('SELECT jid, name, last_message_time FROM chats ORDER BY last_message_time DESC LIMIT 10').all(), null, 2));
\""
```

### Instance Management (via AWS CLI)

```bash
# Check instance state
aws ec2 describe-instances --instance-ids i-0a18ae97ecc4747b5 --query "Reservations[].Instances[].State.Name" --output text --region us-east-2

# Reboot (when SSH is unresponsive)
aws ec2 reboot-instances --instance-ids i-0a18ae97ecc4747b5 --region us-east-2

# Check health
aws ec2 describe-instance-status --instance-ids i-0a18ae97ecc4747b5 --region us-east-2 --query "InstanceStatuses[].[InstanceState.Name,SystemStatus.Status,InstanceStatus.Status]" --output text

# Console log (check for OOM)
aws ec2 get-console-output --instance-id i-0a18ae97ecc4747b5 --region us-east-2 --latest --query "Output" --output text | grep -i "oom\|out of memory\|killed process"

# Resize instance (requires stop first)
aws ec2 stop-instances --instance-ids i-0a18ae97ecc4747b5 --region us-east-2
aws ec2 modify-instance-attribute --instance-id i-0a18ae97ecc4747b5 --instance-type '{"Value":"t3.large"}' --region us-east-2
aws ec2 start-instances --instance-ids i-0a18ae97ecc4747b5 --region us-east-2
```

### Memory & Swap

Instance has 7.6GB RAM (t3.large) + 8GB swap. Previously t3.medium (3.7GB) which caused OOM kills. If SSH becomes unresponsive:

1. Check if instance is running: `aws ec2 describe-instances ...`
2. Check console for OOM: `aws ec2 get-console-output ...`
3. Reboot: `aws ec2 reboot-instances ...`
4. Wait ~75s for SSH to come back

Swap file: `/swapfile` (8GB, persistent via `/etc/fstab`).

### Mount Allowlist

Agent mount paths are validated against `~/.config/nanoclaw/mount-allowlist.json` on the remote host. Format:

```json
{
  "allowedRoots": [
    { "path": "/home/ec2-user/nanoclaw-projects", "allowReadWrite": true },
    { "path": "/home/ec2-user/nanoclaw-tools", "allowReadWrite": true }
  ],
  "blockedPatterns": [],
  "nonMainReadOnly": true
}
```

`allowedRoots` must be objects with `path` and `allowReadWrite` — not plain strings.

### File Locations on Remote Host

| Path | Purpose |
|------|---------|
| `~/nanoclaw/` | NanoClaw source and runtime |
| `~/nanoclaw/.env` | All credentials (gitignored, shadowed from containers) |
| `~/nanoclaw/data/env/env` | Copy of .env for channel startup |
| `~/nanoclaw/groups/telegram_main/` | Main agent workspace (CLAUDE.md, TOOLS.md, procedures/, memory/) |
| `~/nanoclaw/groups/slack_devs/` | Slack #devs workspace |
| `~/nanoclaw/groups/slack_kurtis-dm/` | Slack DM workspace |
| `~/nanoclaw/store/messages.db` | SQLite database |
| `~/nanoclaw/logs/` | Service and container logs |
| `~/nanoclaw-projects/` | Persistent projects mount → `/workspace/extra/projects` in container |
| `~/nanoclaw-tools/` | Persistent tools/SDKs mount → `/workspace/extra/tools` in container |
| `~/.onecli/` | OneCLI compose config |
| `~/.gmail-mcp/` | Gmail OAuth credentials |
| `~/.config/nanoclaw/mount-allowlist.json` | Mount validation allowlist |

### GitHub TOTP (Mark's Account)

```bash
python3 -c "
import hmac, base64, struct, time
secret = 'LFRCQ7RIT4566N4T'
key = base64.b32decode(secret, casefold=True)
counter = int(time.time()) // 30
msg = struct.pack('>Q', counter)
h = hmac.new(key, msg, 'sha1').digest()
o = h[19] & 15
code = (struct.unpack('>I', h[o:o+4])[0] & 0x7fffffff) % 1000000
print(f'{code:06d}')
"
```

## Troubleshooting

**Mark not responding:** Check container logs. Common causes: OOM (check `aws ec2 get-console-output`), API credential issues (check OneCLI health), container spawn failure (check `tail logs/nanoclaw.log`).

**SSH timing out:** Instance likely OOM. Reboot via AWS CLI. Never run background SSH commands.

**Retry loop in logs:** Usually a mount validation error or container spawn failure. Check for `WARN` or error messages around the retry entries.

**WhatsApp not connecting after upgrade:** WhatsApp is now a separate skill, not bundled in core. Run `/add-whatsapp` (or `npx tsx scripts/apply-skill.ts .claude/skills/add-whatsapp && npm run build`) to install it. Existing auth credentials and groups are preserved.

## Container Build Cache

The container buildkit caches the build context aggressively. `--no-cache` alone does NOT invalidate COPY steps — the builder's volume retains stale files. To force a truly clean rebuild, prune the builder then re-run `./container/build.sh`.
