# Project: netbird-github-action-test

## Purpose

This repository is a reference implementation for deploying Docker containers to a VPS over SSH where **port 22 is not publicly accessible**. Access is gated through [NetBird](https://netbird.io/), a WireGuard-based mesh VPN. GitHub Actions joins the NetBird network as a temporary peer, establishes a VPN tunnel, then SSHs into the VPS to run deployment commands.

This solves the problem of allowing CI/CD pipelines to reach servers that have no publicly open ports — only whitelisted NetBird peers can connect.

## Architecture

```
┌─────────────────────┐        NetBird VPN Tunnel        ┌──────────────────┐
│  GitHub Actions      │ ◄──────────────────────────────► │  VPS             │
│  (ubuntu-latest)     │    WireGuard encrypted mesh      │  <VPS_PUBLIC_IP>  │
│                      │                                  │                  │
│  1. netbird-connect  │                                  │  - NetBird peer  │
│  2. SSH over tunnel  │──── SSH (port 22, internal) ────►│  - Docker host   │
│  3. docker commands  │                                  │  - Port 22 closed│
└─────────────────────┘                                   │    to public     │
                                                          └──────────────────┘
```

### Network Flow

1. GitHub Actions runner installs NetBird client via `Alemiz112/netbird-connect@v1`
2. Runner authenticates to NetBird management API using a **setup key** (stored as GitHub secret)
3. NetBird assigns the runner a peer identity and establishes WireGuard tunnels to other peers
4. Runner SSHs to the VPS through the VPN overlay (either the VPS's public IP or NetBird IP, depending on routing config)
5. SSH session executes deployment commands on the VPS
6. After the job completes, the runner is destroyed and the peer is automatically removed

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml     # The single GitHub Actions workflow
├── CLAUDE.md              # This file — AI context
├── PLAN.md                # Setup plan with checklists
└── README.md              # User-facing documentation
```

There is no application code in this repo. It is purely a CI/CD pipeline template.

## Workflow: `.github/workflows/deploy.yml`

### Triggers
- **Push to `main`** — every push triggers a deployment
- **Manual dispatch** — can be triggered from Actions tab or `gh workflow run deploy.yml`

### Job: `deploy`

Runs on `ubuntu-latest`. Steps:

| Step | What it does |
|---|---|
| `actions/checkout@v4` | Checks out the repo (needed for workflow context) |
| `Alemiz112/netbird-connect@v1` | Installs NetBird, joins the network using `NETBIRD_SETUP_KEY`. Sets hostname to `gh-actions-<run_id>` for traceability in NetBird dashboard |
| Wait for connection | Sleeps 5s for tunnel establishment, runs `netbird status` for logs, pings VPS to verify connectivity |
| Deploy via SSH | Writes SSH private key to disk, SSHs into VPS, runs deployment commands |

### Current deploy commands

The deploy step currently runs a **smoke test** (prints hostname and docker version). The actual docker compose deployment commands are commented out — uncomment and customize when ready for real deployments.

## GitHub Secrets

All secrets are configured on the repo `khanakia/netbird-github-action-test`:

| Secret | Purpose | Notes |
|---|---|---|
| `NETBIRD_SETUP_KEY` | Authenticates the GitHub Actions runner as a NetBird peer | Created in NetBird Dashboard → Setup Keys. Must be **reusable** type since each workflow run is a new peer. Has an expiry date — must be rotated before it expires. |
| `VPS_HOST` | IP address to SSH into | Currently `<VPS_PUBLIC_IP>` (public IP). Could also be a NetBird overlay IP (`100.x.x.x`) depending on routing. |
| `VPS_USER` | SSH username | Currently `root` |
| `VPS_PORT` | SSH port | Currently `22` |
| `VPS_DEPLOY_KEY` | SSH private key contents | Sourced from local file `vps_deploy.key`. The corresponding public key must be in the VPS's `~/.ssh/authorized_keys`. |

## External Dependencies

### NetBird (https://netbird.io/)
- **What**: WireGuard-based mesh VPN with a management layer
- **Dashboard**: https://app.netbird.io — manage peers, setup keys, ACL policies, groups
- **Setup Keys**: Credentials that let new peers join the network. Can be one-time or reusable. Have expiry dates.
- **ACL Policies**: Control which peer groups can talk to which. The GitHub Actions peer group must be allowed to reach the VPS peer on port 22.
- **Peers**: Each workflow run creates a temporary peer named `gh-actions-<run_id>`. These auto-expire but may accumulate in the dashboard — periodic cleanup may be needed.

### Alemiz112/netbird-connect (GitHub Action)
- **Repo**: https://github.com/Alemiz112/netbird-connect
- **Version pinned**: `v1`
- **Inputs**: `setup-key` (required), `hostname` (optional), `management-url` (optional, defaults to `https://api.netbird.io:443`), `args` (optional extra CLI flags)
- **What it does**: Installs the NetBird client on the runner, runs `netbird up` with the provided setup key, making the runner a peer on the network

### VPS Details
- **Public IP**: `<VPS_PUBLIC_IP>`
- **OS**: Linux (specific distro not documented)
- **SSH user**: `root`
- **Port 22**: Closed to public internet, accessible only via NetBird VPN
- **Docker**: Installed and available
- **NetBird client**: Installed and running as a connected peer

## Key Design Decisions

1. **Reusable setup key over one-time keys**: Each workflow run creates a new peer. One-time keys would require generating a new key per run, adding complexity. Reusable keys are simpler but slightly less secure — acceptable for this use case since the key is in GitHub encrypted secrets.

2. **`StrictHostKeyChecking=no`**: Necessary because the VPS host key isn't pre-known to the ephemeral runner. In a hardened setup, you could pin the host key fingerprint as a secret.

3. **Sleep before status check**: NetBird needs a few seconds after `netbird up` to establish WireGuard tunnels. The 5-second sleep is a pragmatic buffer.

4. **Heredoc for SSH commands**: Using `<< 'DEPLOY'` (quoted) prevents variable expansion on the runner side — all commands execute on the VPS with its own environment.

5. **Hostname includes run ID**: `gh-actions-${{ github.run_id }}` makes it easy to trace which workflow run created which peer in the NetBird dashboard.

## Common Operations

### Trigger a deployment
```bash
gh workflow run deploy.yml --repo khanakia/netbird-github-action-test
```

### Watch a running workflow
```bash
gh run watch --repo khanakia/netbird-github-action-test
```

### Check recent runs
```bash
gh run list --limit 5 --repo khanakia/netbird-github-action-test
```

### View logs of a specific run
```bash
gh run view <run-id> --log --repo khanakia/netbird-github-action-test
```

### Update a secret
```bash
gh secret set SECRET_NAME --body "value" --repo khanakia/netbird-github-action-test
```

### Rotate the NetBird setup key
1. Create a new setup key in NetBird Dashboard
2. `gh secret set NETBIRD_SETUP_KEY --body "NEW-KEY-HERE" --repo khanakia/netbird-github-action-test`

## Gotchas and Failure Modes

- **Setup key expired**: NetBird setup keys have expiry dates. If the workflow suddenly fails at the "Connect to NetBird" step, check the key's expiry in the dashboard.
- **Stale peers**: Each workflow run registers a new peer. If you run many deployments, the NetBird dashboard may accumulate stale peers. These don't affect functionality but are noisy.
- **ACL misconfiguration**: If you change NetBird groups or policies, the GitHub Actions peer may lose access to the VPS. Check ACL rules if SSH times out after NetBird connects successfully.
- **SSH key mismatch**: If `VPS_DEPLOY_KEY` doesn't match what's in the VPS's `authorized_keys`, SSH auth will fail. The workflow will show "Permission denied (publickey)".
- **NetBird tunnel not ready**: The 5-second sleep may not be enough on slow connections. If ping/SSH fails intermittently, increase the sleep or add a retry loop.
