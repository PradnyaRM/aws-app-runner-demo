# Manual Deployment Steps — AWS App Runner PoC

## Prerequisites

- AWS CLI installed and configured (`aws configure`)
- GitHub account with this repo pushed to it
- AWS account with permissions for: App Runner, WAF, IAM

> No Docker required — App Runner builds directly from GitHub source.

---

## Step 1 — Push Repo to GitHub

```bash
git init
git add .
git commit -m "initial poc"
git remote add origin https://github.com/<your-org>/aws-app-runner-demo.git
git push -u origin main
```

---

## Step 2 — Connect GitHub to App Runner (one-time)

1. Go to **AWS Console → App Runner → GitHub connections**
2. Click **Add new** → authenticate with GitHub
3. Authorize AWS Connector for GitHub (installs as a GitHub App on your org/repo)
4. Connection will show as **Available** once done

---

## Step 3 — Create App Runner Service (Console)

1. Go to **AWS Console → App Runner → Create service**
2. Source: **Source code repository**
3. Connect to your GitHub repo: `<your-org>/aws-app-runner-demo`
4. Branch: `main`
5. Deployment trigger: **Automatic** (deploys on every push to main)
6. Runtime: **Python 3**
7. Build settings: **Use configuration file** → App Runner will pick up `apprunner.yaml`
8. Service settings:
   - Service name: `app-runner-poc`
   - CPU: `0.25 vCPU`
   - Memory: `0.5 GB`
   - Health check path: `/health`
9. Click **Create & deploy** and wait ~3-5 minutes
10. Copy the generated **App Runner HTTPS URL**

---

## Step 4 — Attach WAF IP Allowlist (Console)

> This restricts access to Pivotree staff IPs only.

**First — Create the IP Set**

1. Go to **AWS Console → WAF & Shield → IP sets**
2. Click **Create IP set**
   - Name: `pivotree-staff-ips`
   - Region: `us-east-1` (must match App Runner region)
   - IP version: IPv4
   - Add Pivotree office/VPN CIDR ranges (e.g. `52.15.149.168/32`)
3. Click **Create IP set**

**Then — Create the Web ACL**

4. Go to **AWS Console → WAF & Shield → Web ACLs**
5. Click **Create Web ACL**
   - Name: `app-runner-poc-acl`
   - Region: `us-east-1`
   - Resource type: **App Runner service**
6. **Add rules → Add my own rules → IP set rule**
   - Select the `pivotree-staff-ips` IP set you just created
   - Rule action: **Allow**
7. **Default action: Block** (everything not in the IP set)
8. Click through and **Create Web ACL**
9. Associate the Web ACL with your `app-runner-poc` service

---

## Step 5 — Verify

```bash
# Should return 200 with "Hello from AWS App Runner"
curl https://<your-app-runner-url>

# Health check endpoint
curl https://<your-app-runner-url>/health

# Test WAF block — connect via a non-allowed IP (e.g. mobile hotspot), should return 403
```

---

## Step 6 — Check Logs (CloudWatch)

1. Go to **AWS Console → CloudWatch → Log groups**
2. Find log group: `/aws/apprunner/app-runner-poc/<service-id>/application`
3. Observe structured log output from the app

---

## Step 7 — Test Auto-Deploy (Bonus)

1. Edit `app.py` — change the response message
2. Push to `main`
3. Watch App Runner automatically detect the change and redeploy (~3-5 mins)

---

## Cost Estimate

| Resource | Pricing | Estimated Monthly |
|---|---|---|
| App Runner (0.25 vCPU) | $0.064/vCPU-hr | ~$11.52 (always on) |
| App Runner (0.5 GB RAM) | $0.007/GB-hr | ~$2.52 (always on) |
| App Runner build | $0.005/build-min | Minimal for PoC |
| WAF Web ACL | $5.00/month base | $5.00 |
| WAF requests | $0.60/million | Minimal for PoC |
| **Total estimate** | | **~$19–20/month** |

> Note: App Runner scales to zero when idle if auto-scaling is configured, which can significantly reduce compute costs.

---

## Observations

| Area | Notes |
|---|---|
| Ease of setup | Straightforward via console. Source-based deployment (GitHub → App Runner) requires no Docker locally. One gotcha: WAF IP set must be created before the Web ACL, otherwise the console blocks you with a required field error. |
| Required AWS knowledge | Moderate. Need familiarity with IAM roles, WAF concepts, and CloudWatch. GitHub connection OAuth flow is simple. App Runner itself is beginner-friendly. |
| Deployment speed | ~3-5 mins from push to live URL. First deploy took ~4 mins including build. Auto-deploy on push works reliably. |
| Logging/visibility | CloudWatch log groups created automatically. Application stdout is streamed with no extra config needed. |
| Access restriction | WAF IP allowlist works well. Not a native security group — operates at HTTP layer (Layer 7), returns 403 for blocked IPs. Sufficient for staff-only access. |
| Cost accuracy | Aligned with estimates (~$19-20/month always-on). Can be reduced significantly if auto-scaling to zero is enabled. |

## ⚠️ Deprecation Notice

> **AWS announced that App Runner will no longer accept new customers starting April 30, 2026.** Existing services remain operational but no new features will be added. AWS recommends migrating to **Amazon ECS Express Mode** for new containerized workloads.

**Recommendation:** Do not adopt App Runner for new production workloads. This PoC is valid as a learning exercise and for evaluating the migration path to ECS Express Mode.
