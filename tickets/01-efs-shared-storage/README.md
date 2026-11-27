# Ticket 01: Shared storage for two app servers, with access control

> **Request:** "Two app servers need shared files, and the team lead wants to know who can read what."

**Status:** Done · **Date:** 2026-11-19 · **Region:** us-east-1

## Result

Two EC2 instances share one **Amazon EFS** file system. Access is controlled at two layers:

1. **Machine level (IAM):** only instances with the `ec2-efs-role` role can mount, and only over **TLS**.
2. **User level (Linux):** inside the share, normal owner/group permissions protect each user's files.

`root` on a server is **squashed** (treated as an anonymous user), so `sudo` on a server doesn't give root over the shared files.

**One gap found:** NFS identifies users only by their **UID number**, so a different user given alice's UID could read her files. The production fix is EFS access points (below).

## How it works

```
 Public subnet (1a)                      Private subnet (1b)
 efs-lab-instance1 ──── SSH (agent fwd) ──▶ efs-lab-instance2
 role: ec2-efs-role                         role: ec2-efs-role
        │                                          │
        └──── NFS over TLS (2049) ──▶ Amazon EFS ◀─┘
                                     (mount target per AZ)

 Private route table ── S3 gateway endpoint ──▶ Amazon Linux package repos
```

- The **file system policy** (on EFS) allows only `ec2-efs-role` and **denies any unencrypted connection**.
- The **role** allows mount and write, but not root access, which is what turns on root squash.
- The private instance installs packages through a free **S3 gateway endpoint**, with no internet access.

## What I built

| Resource | Setting |
| --- | --- |
| VPC | Public subnet (1a) + private subnet (1b), internet gateway |
| `efs-lab-instance1` / `instance2` | Public (SSH entry point) / private (reached by SSH agent forwarding) |
| EFS `fs-0081eb4b40b474964` | Mount target in each subnet; NFS 2049 allowed only from the instances' SGs |
| Role `ec2-efs-role` | `ClientMount` + `ClientWrite` on this file system only, **no `ClientRootAccess`** |
| File system policy | Allow only `ec2-efs-role`; deny if not TLS |
| `amazon-efs-utils` | Mount helper on both instances: `mount -t efs -o tls,iam` |
| S3 gateway endpoint | Package installs for the private instance |
| Linux users | `alice` and `bob` on both servers with matching UIDs, owner-only private folders |

## Tests

| # | Test | Result | Evidence |
| --- | --- | --- | --- |
| 1 | Instance 2 (no role): plain NFS mount | Denied | `01` |
| 2 | Instance 2 (no role): TLS mount without IAM | Denied | `02` |
| 3 | Instance 2 (no role): TLS + IAM mount | Fails, no credentials | `03` |
| 4 | Instance 1 (role): plain / TLS only / TLS + IAM | Denied / denied / **mounted** | `04` |
| 5 | Instance 1: `sudo touch` on EFS | Denied (root squash) | `05` |
| 6 | Instance 2 with role: TLS + IAM, read file written on Instance 1 | **Mounted, file visible** | `06` |
| 7 | `bob` reads alice's folder | Denied on both servers | — |
| 8 | `eve`, given **alice's UID**, reads alice's folder | **Allowed** (the gap) | — |

Evidence numbers match the files in [`screenshots/`](screenshots/).

## Who can read what

| Who | Mount EFS? | Read alice's folder? | Why |
| --- | --- | --- | --- |
| Instance with `ec2-efs-role` | Yes, TLS only | Depends on Linux user | Allowed by role + file system policy |
| Instance without the role | No | No | No IAM identity |
| Any plain NFS client | No | No | Explicit deny on unencrypted transport |
| `root` / `sudo` | Mounts, but acts as "nobody" | No | Root squash |
| `alice` | — | Yes | Owner |
| `bob` | — | No | Folder is owner-only |
| Anyone with alice's UID | — | **Yes** | NFS trusts the UID the client sends |

## Problems I hit

| Problem | Fix |
| --- | --- |
| `eve` with alice's UID read alice's folder | Limited mounting to trusted instances (IAM) + root squash. Full fix: EFS access points |
| SSH hop to Instance 2: `Permission denied (publickey)` | Reconnected with `ssh -A` (agent forwarding) |
| Private instance couldn't install `amazon-efs-utils` | Added an S3 gateway endpoint instead of a paid NAT gateway |
| Old plain NFS mounts stopped working | Expected after the TLS policy. Switched to `-o tls,iam` |

## In production

- **EFS access points** that force a fixed UID/GID per app, so the client's UID is ignored (closes the gap).
- One role per app, each allowed only its own access point.
- Each subnet gets its own named route table (not the VPC's main one).
- Restrict the S3 endpoint policy to the Amazon Linux repo buckets.
- Persistent mounts in `/etc/fstab` with `_netdev,tls,iam`.
- Build it all in Terraform.

## Key takeaways

- NFS trusts the client completely. Real access control needs a layer above the UID.
- An identity policy (role) and a resource policy (file system) work together. The resource policy keeps everyone else out.
- An explicit `Deny` always beats an `Allow`.
- VPC endpoints let private servers reach AWS services without internet access.

<details>
<summary><b>Request flow (click to expand)</b></summary>

| # | Who → who | Port | How it's authenticated |
| --- | --- | --- | --- |
| 1 | Laptop → Instance 1 | 22 | SSH key, with agent forwarding (`-A`) |
| 2 | Instance 1 → Instance 2 (private IP) | 22 | Same key, forwarded by the agent; never copied to the server |
| 3 | `efs-utils` → instance metadata (IMDS) | 80, link-local | Returns `ec2-efs-role` temporary credentials |
| 4 | Instance → EFS mount target | 2049 over TLS | IAM-signed mount as `ec2-efs-role`; file system policy checks the role and `aws:SecureTransport` |
| 5 | File reads/writes on EFS | 2049 over TLS | Linux permissions by UID/GID; root mapped to "nobody" |
| 6 | Instance 2 → package repos via S3 gateway endpoint | 443 | Route table sends S3 traffic to the endpoint; no internet path |

</details>

<details>
<summary><b>Revision notes (click to expand)</b></summary>

**File system policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLabRoleMountAndWrite",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/ec2-efs-role" },
      "Action": ["elasticfilesystem:ClientMount", "elasticfilesystem:ClientWrite"],
      "Resource": "arn:aws:elasticfilesystem:us-east-1:<ACCOUNT_ID>:file-system/fs-0081eb4b40b474964"
    },
    {
      "Sid": "DenyUnencryptedTransport",
      "Effect": "Deny",
      "Principal": { "AWS": "*" },
      "Action": "*",
      "Resource": "arn:aws:elasticfilesystem:us-east-1:<ACCOUNT_ID>:file-system/fs-0081eb4b40b474964",
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    }
  ]
}
```
- Statement 1: only this role can mount and write. No `ClientRootAccess`, so root is squashed.
- Statement 2: anyone connecting **without TLS** is denied, and an explicit Deny beats any Allow.

**The three EFS actions**
| Action | Allows |
| --- | --- |
| `ClientMount` | Mount (read-only on its own) |
| `ClientWrite` | Write |
| `ClientRootAccess` | Act as root on the file system. Leave it out → root squash |

**Commands**
```bash
sudo yum install -y amazon-efs-utils                           # EFS mount helper
sudo mount -t efs -o tls,iam fs-0081eb4b40b474964:/ /mnt/efs   # mount with TLS + IAM
df -h | grep efs                                               # confirm it's mounted
sudo umount /mnt/efs                                           # unmount
ssh -A ec2-user@<instance1-public-ip>                          # connect with agent forwarding
```

**Why the UID gap exists:** NFS sends the user's UID number with each request, and EFS trusts it. Two servers with matching UIDs share permissions, but so does anyone who can create a user with the same number. Access points fix this by overriding the UID on the server side.

**AWS credential lookup order:** environment variables → credentials file → config file → container credentials → instance role. That's why an instance role works without stored keys, and why stray keys on a server would quietly take priority.

**S3 gateway endpoint vs NAT gateway:** the gateway endpoint is free and only reaches S3 (where Amazon Linux repos live). A NAT gateway costs per hour and opens the whole internet.

</details>
