#  Universal CI/CD Playbook (A–Z)

> **Author:** Uzair
> **Purpose:** General‑purpose CI/CD reference for all future projects
> **Scope:** Backend + Frontend, Docker / Non‑Docker, Low‑disk servers, Restricted SSH

---

##  Core Principles (Non‑Negotiable)

1. **CI validates code, CD deploys code**
2. **Never build frontend on low‑disk production servers**
3. **Never assume Docker is installed**
4. **Never assume SSH is publicly accessible**
5. **Uploads ≠ Source code**
6. **Always check disk before debugging anything else**

---

##  Standard Repository Structure

```
repo/
├── backend/
│   └── Dockerfile (optional)
├── frontend/
│   └── dist/ (generated in CI)
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
```

---

##  Branch Strategy

```
dev   → development
main  → production
```

* CI runs on `dev`
* CI runs on PR → `main`
* CD runs **only** on `main`

---

#  SSH KEY MANAGEMENT (FULL FLOW)

This section is **critical** and was missing before.

---

##  OPTION A: Generate SSH Key **ON SERVER** (Recommended for Deploy Keys)

### 1️ Login to server

```bash
ssh ubuntu@SERVER_IP
```

---

### 2️ Generate SSH key on server

```bash
ssh-keygen -t ed25519 -C "github-deploy"
```

Press **Enter** for default path:

```
/home/ubuntu/.ssh/id_ed25519
```

(No passphrase recommended for CI)

---

### 3️ Verify keys

```bash
ls ~/.ssh
cat ~/.ssh/id_ed25519.pub
```

You will see something like:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... github-deploy
```

---

### 4️ Add PUBLIC key to GitHub **Deploy Keys**

📍 GitHub Repo → **Settings** → **Deploy Keys** → **Add deploy key**

* Title: `prod-server-key`
* Paste **id_ed25519.pub**
* ✅ Check **Allow write access** (required for pull)

---

### 5️ Test GitHub access from server

```bash
ssh -T git@github.com
```

✅ Expected:

```
Hi <username>! You've successfully authenticated...
```

❌ If you get `Permission denied (publickey)` → key not added correctly

---

##  OPTION B: GitHub Actions → Server (CI/CD Push)

### 1️ Generate key LOCALLY or in CI machine

```bash
ssh-keygen -t ed25519 -C "github-actions"
```

---

### 2️ Add PUBLIC key to server

```bash
nano ~/.ssh/authorized_keys
```

Paste public key

Set permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

### 3️ Add PRIVATE key to GitHub Secrets

GitHub → Repo → Settings → Secrets → Actions

```
SSH_PRIVATE_KEY
SSH_HOST
SSH_USER
```

---

### 4️ Verify SSH from GitHub runner

```yaml
- name: Test SSH
  run: ssh -o StrictHostKeyChecking=no $SSH_USER@$SSH_HOST "echo ok"
```

---

##  CI WORKFLOW (ci.yml)

Purpose: **Build + Validate only**

```yaml
name: CI

on:
  push:
    branches: [dev]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Backend
        working-directory: backend
        run: npm ci

      - name: Frontend
        working-directory: frontend
        run: |
          npm ci
          npm run build
```

---

##  CD WORKFLOW (deploy.yml)

Runs **ONLY** on `main`

### Strategy

* Build frontend in CI
* Rsync `dist/` to server
* Restart backend service

---

##  Server Disk Reality Check

### Always run BEFORE deploy

```bash
df -h
```

If `/` > 85% → STOP

---

### Identify disk hogs

```bash
sudo du -h --max-depth=1 /var | sort -hr
```

---

### Emergency cleanup

```bash
sudo journalctl --vacuum-size=50M
sudo apt clean
rm -rf node_modules dist
```

---

##  Docker Rules (Low Disk Servers)

❌ Do NOT pull large images
❌ Do NOT build frontend in Docker

✔ If Docker used:

```bash
docker system prune -af --volumes
```

---

##  Uploads Folder Rule

* ❌ Never commit uploads
* ❌ Never deploy uploads via CI/CD

`.gitignore`

```
uploads/
public/uploads/
```

Uploads live **only on server**

---

##  Common Errors & Meaning

| Error                         | Real Cause           |
| ----------------------------- | -------------------- |
| no space left                 | Disk full            |
| Permission denied (publickey) | Wrong key            |
| Host key verification failed  | known_hosts missing  |
| docker: no space              | /var/lib/docker full |

---

##  Final Golden Rules

1. SSH first, CI second
2. Disk first, Docker later
3. Build frontend in CI, not prod
4. Uploads are runtime data
5. Simpler pipeline = fewer outages

---

## ✅ Pre‑Project Checklist

* [ ] Disk size checked
* [ ] SSH verified
* [ ] Deploy keys added
* [ ] Uploads ignored
* [ ] CI tested
* [ ] CD dry‑run done

---

**This document is production‑tested.**
