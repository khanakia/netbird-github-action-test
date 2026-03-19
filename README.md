# Deploy to VPS via NetBird VPN

Deploy Docker containers to a VPS over SSH using [NetBird VPN](https://netbird.io/) from GitHub Actions — no publicly open SSH port required.

## Why?

Exposing port 22 to the internet is a security risk. With NetBird, the VPS SSH port is only accessible to peers on your private network. This workflow makes GitHub Actions a temporary NetBird peer, SSHs into the VPS through the encrypted tunnel, and deploys your Docker containers.

## How It Works

```
GitHub Actions Runner
    │
    ├── 1. Joins NetBird network (via setup key)
    ├── 2. Establishes VPN tunnel to your VPS
    ├── 3. SSHs into VPS through the tunnel
    └── 4. Runs deployment commands (docker compose, etc.)
```

The workflow uses [Alemiz112/netbird-connect](https://github.com/Alemiz112/netbird-connect) to join the NetBird network, then connects to your VPS over SSH through the VPN overlay.

## Prerequisites

- **VPS**: NetBird client installed and running, Docker installed, SSH key auth configured
- **NetBird Dashboard**: A setup key created with access to reach the VPS peer

## Setup

### 1. Create a NetBird Setup Key

1. Go to [NetBird Dashboard](https://app.netbird.io) → **Setup Keys** → **Create Setup Key**
2. Name it `github-actions`
3. Set type to **Reusable** (each workflow run creates a new peer)
4. Enable **Ephemeral** (important — see below)
5. Set expiry as needed
6. Assign it to a group that has network access to your VPS peer
7. Make sure your ACL policies allow this group to reach the VPS on port 22

> **Why Ephemeral?** Every workflow run registers a new peer on your NetBird network. Without ephemeral, these peers pile up as stale entries in your dashboard. When a setup key is marked **Ephemeral**, NetBird automatically removes the peer after ~10 minutes of inactivity — perfect for CI/CD since the GitHub Actions runner is destroyed after each job. No manual cleanup needed.

### 2. Add GitHub Secrets

Go to your repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret | Description |
|---|---|
| `NETBIRD_SETUP_KEY` | Setup key from NetBird Dashboard |
| `VPS_HOST` | IP address of your VPS (public or NetBird IP depending on your setup) |
| `VPS_USER` | SSH username (e.g., `root`) |
| `VPS_PORT` | SSH port (e.g., `22`) |
| `VPS_DEPLOY_KEY` | Contents of your SSH private key file |

### 3. Customize the Deploy Step

Edit `.github/workflows/deploy.yml` and replace the deploy commands with your actual deployment:

```yaml
- name: Deploy via SSH
  run: |
    mkdir -p ~/.ssh
    echo "${{ secrets.VPS_DEPLOY_KEY }}" > ~/.ssh/deploy_key
    chmod 600 ~/.ssh/deploy_key
    ssh -o StrictHostKeyChecking=no \
        -o ConnectTimeout=30 \
        -i ~/.ssh/deploy_key \
        -p ${{ secrets.VPS_PORT }} \
        ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }} << 'DEPLOY'
    cd /opt/myapp
    docker compose pull
    docker compose up -d
    docker compose ps
    DEPLOY
```

## Running the Workflow

### Automatic (on push)

The workflow triggers automatically on every push to `main`.

### Manual

1. Go to repo → **Actions** → **Deploy to VPS via NetBird**
2. Click **Run workflow**
3. Select the `main` branch
4. Click **Run workflow**

Or via CLI:

```bash
gh workflow run deploy.yml
```

### Monitoring

Watch the run in real time:

```bash
gh run watch
```

Or check the latest run status:

```bash
gh run list --limit 5
```

## Troubleshooting

### NetBird connection fails
- Verify the setup key is valid and not expired in the NetBird Dashboard
- Check that the setup key's group has proper ACL rules to reach the VPS
- Look at the `netbird status` output in the workflow logs

### SSH connection times out
- Confirm the VPS has NetBird running: `netbird status` on the VPS
- Check that the `VPS_HOST` secret matches the correct IP
- Verify the SSH key is authorized on the VPS (`~/.ssh/authorized_keys`)
- Ensure the VPS firewall allows SSH from NetBird interfaces

### SSH authentication fails
- Make sure `VPS_DEPLOY_KEY` contains the **full private key** including `-----BEGIN/END-----` lines
- Verify the corresponding public key is in the VPS `~/.ssh/authorized_keys`

## Security

- VPS port 22 is **closed** to the public internet
- SSH is only reachable through the NetBird VPN overlay
- GitHub Actions joins NetBird temporarily per workflow run and disconnects after
- Ephemeral setup key ensures the peer is automatically deleted after ~10 minutes of inactivity
- Setup key should be scoped to a group with minimal access
- SSH private key is stored as a GitHub encrypted secret, never committed to the repo

## File Structure

```
.
├── .github/workflows/
│   └── deploy.yml     # Deployment workflow
├── PLAN.md            # Detailed setup plan and checklist
└── README.md          # This file
```
