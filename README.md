# AWS App Runner PoC — Pivotree

A simple proof-of-concept demonstrating deployment of a containerized application on AWS App Runner, restricted to Pivotree staff via AWS WAF.

## Repo Structure

```
.
├── app.py            # Sample Python HTTP app (port 8080)
├── apprunner.yaml    # App Runner build & run configuration
├── Dockerfile        # Container definition (optional, not used in source deploy)
├── STEPS.md          # Manual deployment steps (console + CLI)
└── README.md
```

> No Docker required locally — App Runner builds directly from this GitHub repo.

## Deployment

See [STEPS.md](STEPS.md) for full manual deployment steps covering:
- Push repo to GitHub
- Connect GitHub to App Runner
- App Runner service creation (source-based)
- WAF IP allowlist setup
- Verification, logs, and auto-deploy test

## Architecture

```
GitHub (main branch)
   ↓ push triggers build
[App Runner] ← builds using apprunner.yaml
   ↑
[WAF] ← IP allowlist (Pivotree staff CIDRs only)
   ↑
Internet
```

## Next Steps

- [ ] Terraform IaC (S3 backend + CI/CD via GitHub Actions)
- [ ] VPC egress if private AWS resources needed
- [ ] Custom domain + TLS certificate
