# SonarQube Code Analysis

Self-hosted **SonarQube Community Build** for personal code analysis, deployed on the Azure VM (`portfolio-vm`, Ubuntu 24.04) and analyzed from GitHub Actions.

Live at: **https://sonarqube.darjosh.dev**

## Stack

| Component | Details |
|---|---|
| Image | `mc1arke/sonarqube-with-community-branch-plugin:26.5.0.122743-community` |
| SonarQube | Community Build **26.5.0** |
| Branch plugin | [sonarqube-community-branch-plugin](https://github.com/mc1arke/sonarqube-community-branch-plugin) **26.5.0** — enables per-branch & PR analysis on Community (unofficial, unsupported by SonarSource) |
| Database | Shared `azure-vm-postgres-1` (postgres:16-alpine) on the `azure-vm_default` Docker network — same instance serving the app stack |
| Reverse proxy | nginx container (`azure-vm-nginx-1`), vhost `sonarqube.darjosh.dev`, proxied through Cloudflare |
| Memory tuning | web/CE JVMs `-Xmx384m`, search JVM `-Xmx512m` (VM has 3.8 GB RAM) |

## Repository layout

```
.
├── docker-compose.yml           # The exact config deployed to the VM
├── .env.example                 # Template for /opt/apps/sonarqube/.env (never committed)
└── .github/workflows/deploy.yml # CD: apply config changes to the VM
```

## Deploy pipeline

Pushing to `main` (or `workflow_dispatch`) runs the `deploy` workflow: it SSHes into the VM, replaces `/opt/apps/sonarqube/docker-compose.yml` with the repo version, and runs `docker compose up -d`.

Required **repo secrets** (same values as the blink repo):

| Secret | Value |
|---|---|
| `AZURE_VM_SSH_KEY` | - |
| `AZURE_VM_HOST` | - |
| `AZURE_VM_PORT` | - |
| `AZURE_VM_USER` | - |

The `.env` file with `SONAR_DB_PASSWORD` stays on the VM only — it is never shipped by the pipeline.

## How analysis gets there

Consumer repos (e.g. `blink-social-webapp`) run the SonarQube scan in their own GitHub Actions:

```yaml
- uses: SonarSource/sonarqube-scan-action@v7
  with:
    args: >
      -Dsonar.branch.name=${{ github.ref_name }}
      -Dsonar.qualitygate.wait=true
  env:
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

- `SONAR_HOST_URL` = `https://sonarqube.darjosh.dev`
- `SONAR_TOKEN` = a user token generated in SonarQube (My Account → Security → Tokens)
- The branch plugin makes every pushed branch a real branch in the SonarQube UI.

## Maintenance notes

- **First login:** `admin / admin` on a fresh install — change it immediately. The admin password, base URL and tokens live in the SonarQube database (shared postgres), not in this repo.
- **Server base URL** must be set to `https://sonarqube.darjosh.dev` (Administration → Configuration → General Settings → General) — otherwise GitHub App creation and callback URLs break.
- **Backups:** the SonarQube DB is just another database in the shared postgres (`sonarqube`). `pg_dump` it alongside the app databases.
- **Upgrades:** pin the plugin image version. The community branch plugin lags SonarQube releases (currently supports up to 26.5.0), so don't bump the base version until a matching plugin release exists.
- **Cloudflare:** the domain is proxied through Cloudflare. If CI requests start failing with 403 or HTTP/2 stream resets on large downloads, the nginx vhost has `proxy_buffering off` + 600s timeouts already; the definitive workaround is a DNS-only (grey cloud) record for CI traffic.

## Limitations

- Community edition analyzes the main branch only (the branch plugin adds branch support, but PR **decoration** is still Developer+ only).
- The default "Sonar way" gate requires ≥80% coverage on new code and zero new issues — repos without tests will fail the gate once new code accumulates.
