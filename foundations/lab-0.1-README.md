# Lab 0.1 — Stand Up a Daily-Driver Linux Box

**Roadmap phase:** Phase 0 — Workbench Setup
**Author:** Taiwo

---

## Overview

This lab focuses on standing up the base environment every later lab in this roadmap depends on: a live AWS account, a non-root IAM identity, and an EC2 instance used as a daily Linux terminal. The goal was to get comfortable navigating AWS both through the console and the CLI, and to start building real Linux fluency instead of treating the instance as a black box.

---

## What I built

- An AWS account with a zero-spend billing budget alert configured before touching anything else
- A non-root IAM user for daily work, authenticated via `aws login` (short-lived browser-based credentials) instead of a long-term access key
- A t2.micro EC2 instance (Amazon Linux 2023) as a daily-driver terminal
- An SSH connection into the instance from Windows PowerShell using a `.pem` key, with the key locked to read-only via `icacls`

---

## Key concepts learned

- **Console access vs. programmatic access** — human browser login vs. machine/CLI authentication; `aws login` bridges the two using short-lived tokens instead of a permanent access key
- **Security groups** — a virtual firewall; inbound rules control what traffic is allowed *into* an instance, scoped by port, protocol, and source IP
- **Linux permission strings** (`-rwxr--r--`) — file type, then owner/group/other, each with read/write/execute
- **Ownership hierarchy** — `/home` is root-owned and shared; `/home/ec2-user` is the individual, user-owned folder inside it — the same relationship as `C:\Users` vs `C:\Users\<name>` on Windows

---

## Windows vs. Linux tooling

Since I was connecting from Windows PowerShell rather than a native Linux/macOS terminal, a few things needed a Windows-specific approach:

- **File permissions:** Linux uses `chmod`; Windows PowerShell doesn't have it, so `.pem` key permissions were locked down with `icacls` instead
- **SSH client:** PowerShell's built-in OpenSSH client handled the connection once the key permissions were correct

---

## Full notes

📄 Step-by-step troubleshooting log, including every issue hit and how it was resolved: [`documentation.md`](./documentation.md)

---

## Next up

Lab 0.2 — Git and GitHub as the engineering log for this roadmap.


