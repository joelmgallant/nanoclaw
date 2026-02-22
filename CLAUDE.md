# NanoClaw

Personal Claude assistant. See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

Single Node.js process that connects to WhatsApp, routes messages to Claude Agent SDK running in containers (Linux VMs). Each group has isolated filesystem and memory.

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/whatsapp.ts` | WhatsApp connection, auth, send/receive |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations |
| `groups/{name}/CLAUDE.md` | Per-group memory (isolated) |
| `container/skills/agent-browser.md` | Browser automation tool (available to all agents via Bash) |

## Skills

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |

## Development

Run commands directly—don't tell the user to run them.

```bash
npm run dev          # Run with hot reload
npm run build        # Compile TypeScript
./container/build.sh # Rebuild agent container
```

Service management:
```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
```

## VPS Deployment

Host: `ssh qnetwork` (178.156.135.232, root). App runs as `nanoclaw` user.

```bash
# Update deployment
ssh qnetwork
su - nanoclaw
cd /home/nanoclaw/nanoclaw
git pull                                          # or: git reset --hard origin/main (after rebase)
npm install && npm run build
docker build -t nanoclaw-agent:latest container/
sudo systemctl restart nanoclaw
```

```bash
# Check status
ssh qnetwork "systemctl status nanoclaw --no-pager -l"
ssh qnetwork "journalctl -u nanoclaw -f"
```

### WhatsApp Auth Expiry

WhatsApp periodically invalidates linked device sessions. Symptoms: service crash-loops with `WhatsApp authentication required` in logs and high restart counter.

Fix:
```bash
ssh qnetwork
systemctl stop nanoclaw
su - nanoclaw
cd /home/nanoclaw/nanoclaw
rm -rf store/auth && mkdir -p store/auth
npx tsx src/whatsapp-auth.ts --pairing-code --phone 13064508803
# Enter pairing code on phone: WhatsApp → Settings → Linked Devices → Link with phone number
# Or scan the QR code: WhatsApp → Settings → Linked Devices → Link a Device
systemctl start nanoclaw
```

## Container Lifecycle

The container buildkit caches the build context aggressively. `--no-cache` alone does NOT invalidate COPY steps — the builder's volume retains stale files. To force a truly clean rebuild, prune the builder then re-run `./container/build.sh`.
