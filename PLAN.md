# Deploy Docker to VPS via NetBird VPN - GitHub Actions

## Goal

Deploy Docker containers to a VPS over SSH, where port 22 is **not publicly open** and only accessible through [NetBird VPN](https://netbird.io/). GitHub Actions connects to the NetBird network first, then SSHs into the VPS through the VPN tunnel.

## Architecture

```
GitHub Actions Runner
    │
    ├── 1. Join NetBird network (using setup key)
    │
    ├── 2. SSH into VPS via NetBird IP (not public IP)
    │
    └── 3. Deploy Docker containers on VPS
```

## Prerequisites

### On the VPS (<VPS_PUBLIC_IP>)
- [x] NetBird client installed and running (peer connected to your NetBird network)
- [x] Docker & Docker Compose installed
- [x] SSH key-based auth configured for `root` user
- [ ] **Note the NetBird IP** of the VPS (e.g., `100.x.x.x`) — this is what GitHub Actions will SSH into

### On NetBird Dashboard (https://app.netbird.io)
- [ ] Create a **Setup Key** for GitHub Actions (one-time or reusable)
  - Go to: NetBird Dashboard → Setup Keys → Create Setup Key
  - Name: `github-actions`
  - Type: **Reusable** (recommended, since each workflow run creates a new peer)
  - Expiry: Set as needed
  - Auto-groups: Add to a group that has access to the VPS peer
- [ ] Ensure **ACL/policy** allows the GitHub Actions peer group to reach the VPS peer on port 22

### On GitHub (repo: `khanakia/netbird-github-action-test`)
- [ ] Create the repository
- [ ] Add the following **repository secrets**:

| Secret Name | Value | Description |
|---|---|---|
| `NETBIRD_SETUP_KEY` | *(from NetBird dashboard)* | Setup key for joining the NetBird network |
| `VPS_HOST` | `100.x.x.x` | **NetBird IP** of the VPS (NOT the public IP) |
| `VPS_USER` | `root` | SSH user on the VPS |
| `VPS_PORT` | `22` | SSH port |
| `VPS_DEPLOY_KEY` | *(contents of vps_deploy.key)* | SSH private key for authentication |

> **Important**: `VPS_HOST` should be the **NetBird internal IP** (e.g., `100.x.x.x`), not the public IP `<VPS_PUBLIC_IP>`. The public IP has port 22 blocked — the whole point is to route through NetBird.

## Workflow Steps

### Step 1: GitHub Actions joins NetBird network
Using [Alemiz112/netbird-connect](https://github.com/Alemiz112/netbird-connect):
```yaml
- uses: Alemiz112/netbird-connect@v1
  with:
    setup-key: ${{ secrets.NETBIRD_SETUP_KEY }}
    hostname: 'github-actions-${{ github.run_id }}'
```

### Step 2: Wait for NetBird connection & verify
```yaml
- name: Verify NetBird connection
  run: |
    sleep 5
    netbird status
    ping -c 3 ${{ secrets.VPS_HOST }}
```

### Step 3: SSH into VPS and deploy
```yaml
- name: Deploy via SSH
  run: |
    mkdir -p ~/.ssh
    echo "${{ secrets.VPS_DEPLOY_KEY }}" > ~/.ssh/deploy_key
    chmod 600 ~/.ssh/deploy_key
    ssh -o StrictHostKeyChecking=no -i ~/.ssh/deploy_key -p ${{ secrets.VPS_PORT }} ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }} << 'DEPLOY'
      cd /opt/myapp
      docker compose pull
      docker compose up -d
    DEPLOY
```

## File Structure

```
netbird-github-action-test/
├── .github/
│   └── workflows/
│       └── deploy.yml        # Main deployment workflow
├── docker-compose.yml         # Sample app to deploy
├── Dockerfile                 # Sample Dockerfile
└── PLAN.md                    # This file
```

## Setup Checklist

1. [ ] Create GitHub repo `khanakia/netbird-github-action-test`
2. [ ] Find the **NetBird IP** of the VPS (`netbird status` on the VPS, or check NetBird dashboard)
3. [ ] Create a **NetBird Setup Key** from the dashboard
4. [ ] Add all GitHub secrets (listed above)
5. [ ] Push this code to the repo
6. [ ] Trigger the workflow (push to `main` or manual dispatch)
7. [ ] Verify deployment in GitHub Actions logs
8. [ ] Clean up: check NetBird dashboard for stale peers from CI runs

## Security Notes

- The VPS public IP (`<VPS_PUBLIC_IP>`) has port 22 **closed** to the internet
- SSH is only reachable through the NetBird VPN overlay network
- GitHub Actions joins NetBird temporarily per workflow run
- The setup key should be scoped to a specific group with minimal access
- Consider using **one-time setup keys** in production for tighter control
- The SSH deploy key (`vps_deploy.key`) is stored as a GitHub secret, never committed
