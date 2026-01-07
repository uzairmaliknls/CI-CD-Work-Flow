🚀 Universal CI/CD Playbook (Monorepo, Low-Resource Servers, Real-World Constraints)

Author: Uzair
Use Case: Any future backend + frontend project
Tested Against:

Restricted servers (office-only, locked-down)

Low disk (5–10GB)

Docker + Non-Docker setups

React / Angular / Express

GitHub Actions + SSH deploy

📌 Core Principles (READ THIS FIRST)

CI ≠ CD

CI = validate code

CD = deploy code

Never build frontend on small servers

Never assume Docker exists

Never assume SSH is open

Disk space matters more than config

Production servers should RUN code, not BUILD code

🧱 Standard Repo Structure (Monorepo)
repo/
├── api.project.com/        # Backend (Node / Express)
│   └── Dockerfile          # Optional
├── admin.project.com/      # Frontend (React / Angular)
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml

🌿 Branch Strategy (Simple & Safe)
dev   → all developers work here
main  → production only

Rules

CI runs on:

push to dev

pull_request → main

CD runs ONLY on:

push to main (after merge)

🔐 SSH: A–Z Setup & Verification
1️⃣ Generate SSH key (LOCAL machine)
ssh-keygen -t ed25519 -C "github-actions-project"

2️⃣ Add public key to server
nano ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

3️⃣ Add private key to GitHub Secrets
SSH_PRIVATE_KEY
SSH_HOST
SSH_USER

4️⃣ Verify SSH connectivity (IMPORTANT)

From GitHub runner OR local:

ssh ubuntu@SERVER_IP


❌ If timeout → server is restricted
✔️ If connects → usable

🔍 SSH TROUBLESHOOTING MATRIX
Symptom	Meaning	Fix
i/o timeout	Port 22 blocked	Use self-hosted runner or internal network
Permission denied (publickey)	Key mismatch	Check correct private key in secrets
Host key verification failed	known_hosts missing	ssh-keyscan github.com >> ~/.ssh/known_hosts
Works in Termius but not GitHub	Office-only access	❌ GitHub Actions cannot reach server
🧪 CI WORKFLOW (ci.yml)
Purpose

Ensure code builds

Block bad PRs

NO deployment

name: CI - Verify Builds

on:
  push:
    branches: [dev]
  pull_request:
    branches: [main]

jobs:
  backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - working-directory: api.project.com
        run: npm ci

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - working-directory: admin.project.com
        run: |
          npm ci
          npm run build

Angular NOTE
npm ci
npm run build
# OR
npx ng build --configuration production

🚀 CD WORKFLOW (deploy.yml)
Runs ONLY on main
🧠 Deployment Decision Tree (VERY IMPORTANT)
❓ Does server have:
Condition	Decision
< 15GB disk	❌ No Docker builds
SSH blocked externally	❌ No GitHub Actions deploy
Docker missing	Install or skip Docker
Angular frontend	Build on GitHub, not server
✅ RECOMMENDED DEPLOY STRATEGY (LOW DISK)
✔ Build frontend on GitHub
✔ Rsync dist/ to /var/www/html
✔ Backend:

Docker OR

Node + PM2

🐳 Docker Rules (Learned the hard way)
NEVER do this on small servers:
docker pull big-image
npm ci
npm run build

ALWAYS clean Docker
docker system prune -af --volumes

Prevent disk death
docker system prune -f --filter "until=168h"

🧹 Disk Space Survival Checklist
Check space
df -h

Biggest culprits
sudo du -h --max-depth=1 /var | sort -hr

Clean logs (CRITICAL)
sudo journalctl --vacuum-size=50M

Remove junk
rm -rf node_modules dist
npm cache clean --force
sudo apt clean

📦 Uploads Folder (IMPORTANT DECISION)
❌ DO NOT include uploads in repo
✔ Uploads must live on server only

Add to .gitignore:

uploads/
public/uploads/


Reason:

CI/CD overwrites code

Uploads = runtime data

🌐 Nginx Compatibility (SAFE)

Your existing config:

root /var/www/html;
location /api/ {
  proxy_pass http://localhost:3000;
}


✔ No conflict with CI/CD
✔ Rsync to /var/www/html is SAFE
✔ Backend stays untouched

🧨 Common Errors & REAL Meanings
Error	Actual Reason
no space left on device	Disk full (not Docker bug)
address already in use	Old container still running
ng: command not found	Angular CLI not installed
docker: command not found	Docker not installed
Permission denied (publickey)	Wrong SSH key
Host key verification failed	Missing known_hosts
🧩 Golden Rules (Print These)

CI builds, CD deploys

Never build frontend on prod

Always check disk before debugging

SSH first, CI later

Uploads are not source code

Small servers need simple setups

🏁 Final Recommendation (Based on 3 Projects)
Scenario	Best Setup
Restricted server	Self-hosted runner
Low disk (<10GB)	No Docker
Angular/React	Build on GitHub
Express backend	PM2 or light Docker
Office-only access	Internal CI
📎 Future Use Checklist (Before Any New Project)

 Check disk size

 Check SSH accessibility

 Decide Docker or not

 Separate uploads

 CI first, CD later

 Logs cleanup enabled
