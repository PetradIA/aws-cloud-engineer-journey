# Lab 0.1 — Documentation

Full step-by-step notes, issues hit, and how each was resolved.

---

## Problems hit and how I solved them

**1. AWS CLI not recognized in PowerShell**
- Cause: CLI wasn't installed yet
- Fix: Installed via the official MSI installer, reopened PowerShell to pick up the new PATH

**2. IAM console sign-in kept failing ("Authentication failed")**
- Cause: Was manually typing the account ID and guessing the username, both wrong
- Fix: Found the real account ID from the root account menu, and the exact IAM username from the IAM Users list. Also learned `aws login` can bypass manual entry by reusing an active browser console session ("Continue with an active session")

**3. `.pem` key file not found**
- Cause: Assumed the file was in `~/.ssh/`, but it was actually still sitting in Downloads
- Fix: Navigated to `C:\Users\<user>\Downloads`, confirmed with `dir *.pem`, used the exact filename (had a double dash in it: `taiwo--key.pem`)

**4. SSH connection timed out**
- Cause: Home IP address had changed since the security group's inbound SSH rule was created — the rule was scoped to the old IP via `/32` (one exact address only)
- Fix: Checked current IP, compared to the security group rule, updated the inbound rule to the new IP via the "My IP" option in the console
- Note for later: AWS Systems Manager Session Manager avoids this problem entirely by not requiring SSH/open ports at all — a good future upgrade

**5. `cd ~` unexpectedly landed in `/` instead of `/home/ec2-user`**
- Cause: Unclear — possible terminal quirk, never fully diagnosed
- Workaround: Used the full explicit path `cd /home/ec2-user` instead, confirmed with `pwd` each time

**6. `touch myfile.txt`: Permission denied**
- Cause: Was actually sitting in `/home` (root-owned parent directory) rather than `/home/ec2-user` (own home directory) — confirmed with `pwd`
- Fix: `cd /home/ec2-user`, then the file created successfully
- Key lesson: `/home` is the shared, root-owned container for all user folders; `/home/ec2-user` is the individual folder the user actually owns and can write to

---

## Security decision and why

I scoped the SSH inbound rule to my own IP address only, rather than leaving it open to `0.0.0.0/0`. An EC2 instance with port 22 open to the whole internet is a standard target for automated scanning and brute-force attempts within minutes of launch. The tradeoff — the rule breaks whenever my IP changes — is exactly what caused issue #4 above, which is a good argument for eventually moving to AWS Systems Manager Session Manager instead of relying on SSH at all.

---

## Commands learned

| Command | What it does |
|---|---|
| `pwd` | Prints current working directory |
| `ls`, `ls -la` | Lists directory contents; `-la` shows hidden files in long format (permissions, owner, size, date) |
| `cd <path>` | Changes directory |
| `whoami` | Shows which user you're logged in as |
| `echo $HOME` | Prints the system's recorded home directory path for the current user |
| `touch <file>` | Creates an empty file |
| `chmod u+x <file>` | Adds execute permission for the file's owner |
| `ps`, `ps aux` | Shows running processes (`aux` = all users, user-friendly format) |
| `icacls` (PowerShell) | Fixes Windows file permissions so SSH will accept the `.pem` key |

---

## Additional concepts

- **Ports and protocols** — port 22 = SSH, TCP = reliable connection-oriented protocol used for most everyday traffic
- **Special permission bit example:** `/tmp` uses the sticky bit (`t`) — anyone can create files there, but only the file's own owner can delete them

---

## Still open

- Why `cd ~` didn't work as expected — worked around it, didn't root-cause it. Worth revisiting if it happens again.


