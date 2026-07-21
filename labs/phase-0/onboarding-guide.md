# Phase 0 Workbench Setup — Onboarding Guide

This doc walks a new teammate through setting up the exact environment used for this roadmap: an EC2-based Linux workbench, a documented Git/GitHub workflow, and AWS CLI access. Follow it top to bottom and you'll be productive on this account by the end.

---

## 1. AWS Account & Billing Safety

Before touching anything else:
- Confirm you have an AWS account with console access
- Set a **zero-spend billing budget alert** (Billing → Budgets) so you're notified before any charges hit — this should be the very first thing done, not an afterthought

---

## 2. IAM User Setup

Never work under the root account day to day.

- Create (or confirm you have) a dedicated IAM user for daily work
- Current setup uses `AdministratorAccess` for early Phase 0 labs — **this will be scoped down to least-privilege permissions in Phase 1 (Lab 1.2)**. Don't treat this as the long-term standard.
- Note your IAM username and account ID — you'll need both regularly

---

## 3. EC2 — Your Daily-Driver Terminal

- Launch a `t2.micro` EC2 instance (Amazon Linux 2023) — this is free-tier eligible and stays running as your terminal home base for the whole roadmap
- Download the `.pem` key pair at launch time — **do not lose this file**, it's the only way to SSH in
- **Security group:** restrict inbound SSH (port 22) to your own IP address only via the "My IP" option — never leave it open to `0.0.0.0/0`
  - Tradeoff: this rule breaks if your home IP changes (happened during initial setup — see `labs/phase-0/lab-0.1-documentation.md`). A future upgrade is AWS Systems Manager Session Manager, which removes the need for an open SSH port entirely.

**Connecting via SSH (from Windows):**
```powershell
icacls.exe "your-key-name.pem" /reset
icacls.exe "your-key-name.pem" /grant:r "$($env:USERNAME):(R)"
icacls.exe "your-key-name.pem" /inheritance:r
ssh -i "your-key-name.pem" ec2-user@<public-ip>
```
Windows doesn't have `chmod`, so `icacls` is the equivalent way to lock the key file down to read-only before SSH will accept it.

**Common gotcha:** if you land in `/home` and get `Permission denied` creating files, you're in the shared root-owned parent directory, not your own home folder. Run `cd /home/ec2-user` (or `cd ~`) to fix it.

---

## 4. Git & GitHub — The Engineering Log

Every lab in this roadmap gets documented and committed here: [`aws-cloud-engineer-journey`](https://github.com/petraia/aws-cloud-engineer-journey)

**Initial setup on the EC2 box:**
```bash
git config --global user.name "petraia"
git config --global user.email "your-email@example.com"

git clone https://github.com/petraia/aws-cloud-engineer-journey.git
cd aws-cloud-engineer-journey
```

**`.gitignore` — set this up before your first real commit:**
```bash
cat > .gitignore << 'EOF'
*.pem
*.env
.env
terraform.tfstate
terraform.tfstate.backup
.terraform/
*.tfvars
