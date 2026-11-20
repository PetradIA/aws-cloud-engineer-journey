# Ticket 01: Shared storage for two app servers, with access control

> **Request:** "Two app servers need shared files, and the team lead wants to know who can read what."

**Status:** Done · **Date:** 2026-11-19 · **Region:** us-east-1 · **Account:** lab (old account)

## Summary

Two EC2 instances now share one Amazon EFS file system. Access is controlled at two layers:

1. **Machine level (IAM):** only instances with the `ec2-efs-role` IAM role can mount the file system, and only over an encrypted (TLS) connection.
2. **User level (Linux):** inside the share, files and folders are protected by normal Linux owner/group permissions, based on user IDs (UIDs).

Root on a client is "squashed" (mapped to an anonymous user), so gaining `sudo` on a server does not give root over the shared storage.

## Architecture

```
                 Internet
                    │
             Internet Gateway
                    │
┌──────────── VPC (efs-lab) ───────────────────────────────┐
│                                                          │
│  Public subnet (us-east-1a)     Private subnet (us-east-1b)
│  ┌─────────────────────┐        ┌─────────────────────┐  │
│  │ efs-lab-instance1   │  SSH   │ efs-lab-instance2   │  │
│  │ (bastion + app)     │──────▶ │ (app, no public IP) │  │
│  │ role: ec2-efs-role  │        │ role: ec2-efs-role  │  │
│  └─────────┬───────────┘        └─────────┬───────────┘  │
│            │ NFS over TLS (2049)          │              │
│            └──────────▶  Amazon EFS  ◀────┘              │
│                fs-0081eb4b40b474964                      │
│                (mount target per AZ)                     │
│                                                          │
│  Private route table ── S3 gateway endpoint ──▶ Amazon S3│
│  (package installs for Instance 2, no internet)          │
└──────────────────────────────────────────────────────────┘
```

## What I built

| Component | Detail |
| --- | --- |
| VPC | Public subnet (us-east-1a) and private subnet (us-east-1b), Internet Gateway, public route table |
| EC2 | `efs-lab-instance1` (public, SSH entry point) and `efs-lab-instance2` (private, reached by SSH agent forwarding) |
| EFS | `fs-0081eb4b40b474964`, mount targets in both subnets, NFS port 2049 allowed only from the instances' security groups |
| IAM role | `ec2-efs-role`: `elasticfilesystem:ClientMount` + `ClientWrite` on this file system only (no `ClientRootAccess`) |
| EFS file system policy | Allows only `ec2-efs-role`; denies any connection where `aws:SecureTransport` is false |
| Mount helper | `amazon-efs-utils` on both instances, mounted with `mount -t efs -o tls,iam` |
| S3 gateway endpoint | Lets the private instance install packages from Amazon Linux repos (hosted in S3) with no internet access |
| Linux users | `alice` and `bob` created on both instances with matching UIDs, private folders with owner-only permissions |

## Tests and evidence

| # | Instance | Test | Expected | Result | Evidence |
| --- | --- | --- | --- | --- | --- |
| 1 | Instance 2 (no role) | Plain NFS mount | Denied | `access denied by server` | `01-instance2-plain-nfs-denied.png` |
| 2 | Instance 2 (no role) | Helper mount, TLS only | Denied (no IAM identity) | `access denied by server` | `02-instance2-tls-no-iam-denied.png` |
| 3 | Instance 2 (no role) | Helper mount, TLS + IAM | Fails (no credentials) | Credentials not found | `03-instance2-tls-iam-no-role-creds-missing.png` |
| 4 | Instance 1 (role) | Plain NFS, TLS only, TLS + IAM | Fail, fail, success | Fail, fail, mounted | `04-instance1-nfs-denied-tls-denied-tls-iam-mounted.png` |
| 5 | Instance 1 (role) | `sudo touch` on EFS root | Denied (root squash) | `Permission denied` | `05-instance1-root-squash-denied.png` |
| 6 | Instance 2 (role attached) | Helper mount, TLS + IAM, read shared file | Success | Mounted, read `test.txt` written from Instance 1 | `06-instance2-with-role-mounted-reads-shared-file.png` |
| — | Both | User `bob` reads `alice`'s private folder | Denied | Denied on both servers | (Nov 18 session) |
| — | Both | User `eve` with **alice's UID** reads alice's folder | Should be denied | **Allowed** — see finding below | (Nov 18 session) |

## Who can read what

| Who | Can mount EFS? | Can read/write alice's folder? | Why |
| --- | --- | --- | --- |
| Instance with `ec2-efs-role` | Yes (TLS only) | Depends on Linux user | Allowed by IAM + file system policy |
| Instance without the role | No | No | No IAM identity, policy has no allow for it |
| Any client over plain NFS | No | No | Explicit deny on unencrypted transport |
| `root` / `sudo` on a client | Mount yes, but acts as "nobody" | No | Root squash (no `ClientRootAccess`) |
| `alice` | — | Yes | Owner of the folder |
| `bob` | — | No | Not owner, folder is owner-only |
| Any user with alice's UID | — | **Yes** | NFS trusts the UID number the client sends |

## What went wrong and how I fixed it

| Problem | Cause | Fix |
| --- | --- | --- |
| A different user (`eve`) given alice's UID could read alice's private folder | NFS identifies users by UID number only; it trusts whatever the client sends | Added IAM control at machine level so only trusted instances can mount at all (root squash also stops root impersonation). Full fix in production: EFS access points (see below) |
| `ssh` from Instance 1 to Instance 2 failed with `Permission denied (publickey)` | Connected to Instance 1 without agent forwarding (`-A`), so no key was available for the hop | Reconnected with `ssh -A`; added `ForwardAgent yes` in `~/.ssh/config` |
| `amazon-efs-utils` not installed on Instance 2 and no way to install it | Private subnet has no route to the internet | Created an S3 gateway endpoint on the private route table (free, private, S3-only) instead of a NAT gateway |
| Old plain NFS mounts stopped working after the policy | Expected: the policy now requires TLS and IAM | Switched to `mount -t efs -o tls,iam` |

## What I'd do in production

- **EFS access points:** give each application its own access point that forces a fixed UID/GID and a root folder. The client-sent UID is then ignored, which closes the eve gap for real.
- **Scope the role per app:** one IAM role per application, each allowed only its own access point (`elasticfilesystem:AccessPointArn` condition).
- **Dedicated route tables:** the private subnet currently uses the VPC's main route table. In production, leave the main table empty and give each subnet its own named table.
- **Endpoint policy:** restrict the S3 gateway endpoint to the Amazon Linux repo buckets only.
- **Persistent mounts:** add the mount to `/etc/fstab` (with `_netdev,tls,iam`) so it survives reboots.
- **Infrastructure as code:** rebuild all of this in Terraform so it deploys identically every time.

## Key commands

```bash
# Install the EFS mount helper
sudo yum install -y amazon-efs-utils

# Mount with encryption + IAM
sudo mount -t efs -o tls,iam fs-0081eb4b40b474964:/ /mnt/efs

# Check the mount
df -h | grep efs

# Unmount
sudo umount /mnt/efs
```

## File system policy used

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

## What I learned

- NFS on its own trusts the client completely. Real access control on shared storage needs a layer above the UID.
- An identity policy (on the role) and a resource policy (on the file system) work together; the resource policy is what keeps everyone else out.
- An explicit `Deny` always wins over an `Allow`.
- Private instances can reach specific AWS services privately through VPC endpoints, without opening them to the internet.
- The AWS credential lookup order (credentials file → config → container → instance role) explains why instance roles beat stored access keys.
