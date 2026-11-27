# Ticket 05: Shell access to a private server with no SSH

> **Request:** "Security flagged SSH open and keys shared between engineers. Remove SSH, but keep shell access, and make every session traceable to a named person."

**Status:** Done · **Date:** 2026-11-26 · **Region:** us-east-1

## Result

A private server with **no public IP, no SSH key, no inbound rules and no internet access** is reachable through **AWS Systems Manager Session Manager**. Every session is logged under the IAM user who opened it.

Testing the audit trail caught a real problem: the first session was opened as **root**. I fixed it with a least-privilege user.

## How it works

```
Engineer (IAM user) ──▶ Systems Manager
                              ▲
                              │ channel opened OUTBOUND by the agent
                              │
 Private subnet:  SSM endpoints (10.0.2.121, 10.0.2.7) ◀──443── ssm-lab-instance
                  SG: 443 from instance SG only               SG: no inbound at all
```

- The **SSM Agent** on the server connects **out** on port 443, so nothing needs to be open inbound.
- The server has no internet, so three **VPC interface endpoints** (`ssm`, `ssmmessages`, `ec2messages`) bring Systems Manager into the VPC.
- **Private DNS** makes the normal AWS service names resolve to those private endpoint IPs. It needs the VPC's *DNS resolution* and *DNS hostnames* settings on.
- The server proves its identity with an **IAM role** (`AmazonSSMManagedInstanceCore`). People are allowed in by **IAM**, not SSH keys.

*SSM = Simple Systems Manager, the old name of AWS Systems Manager. Session Manager is one tool inside it.*

## What I built

| Resource | Setting |
| --- | --- |
| `ssm-instance-sg` | **No inbound.** Outbound: HTTPS 443 to the endpoint SG only |
| `ssm-endpoint-sg` | Inbound HTTPS 443 from `ssm-instance-sg` |
| 3 interface endpoints | `ssm`, `ssmmessages`, `ec2messages`, private subnet, private DNS on |
| Role `ec2-ssm-role` | `AmazonSSMManagedInstanceCore` |
| `ssm-lab-instance` | Amazon Linux 2023, private subnet, **no key pair**, no public IP, IMDSv2 |
| User `ssm-operator` | Can start sessions on **this one instance only** (policy below) |

## Tests

Evidence numbers match the files in [`screenshots/`](screenshots/).

| Test | Result | Evidence |
| --- | --- | --- |
| Server registers with Systems Manager | Online | `01` |
| Open a session | Logged in as `ssm-user`, no SSH | `03` |
| SSM names resolve privately | `ssm` → `10.0.2.121`, `ssmmessages` → `10.0.2.7` | `04`, `05` |
| Shell travels over the endpoint | Connections to `10.0.2.7:443` | `04` |
| No internet | `curl google.com` times out | `06` |
| Who opened the session? | **root** (finding) | `07` |
| `ssm-operator` tries other things | `Access denied` everywhere else | `09`, `10` |
| `ssm-operator` opens a session | Logged as `ssm-operator`, not root | `11` |

## Findings

| Finding | Status |
| --- | --- |
| Session opened as **root**: can't be restricted, and isn't a named person | **Fixed**: least-privilege `ssm-operator` user |
| Admin user `taiwo-admin` has **no MFA** (`08`) | Open |
| `taiwo-admin` unused for 113 days (`08`) | Open |

## Least-privilege policy (`ssm-operator`)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ssm:StartSession",
      "Resource": [
        "arn:aws:ec2:us-east-1:<ACCOUNT_ID>:instance/i-08ec76a6aa77f2a77",
        "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:document/SSM-SessionManagerRunShell"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:TerminateSession", "ssm:ResumeSession"],
      "Resource": "arn:aws:ssm:*:*:session/${aws:username}-*"
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:DescribeSessions", "ssm:GetConnectionStatus", "ssm:DescribeInstanceInformation", "ssm:DescribeInstanceProperties", "ec2:DescribeInstances"],
      "Resource": "*"
    }
  ]
}
```

- Statement 1: start sessions on **one instance only**.
- Statement 2: `${aws:username}` becomes the caller's name, so users can only end **their own** sessions.
- Statement 3: read-only lookups the console needs.

## Problems I hit

| Problem | Fix |
| --- | --- |
| Couldn't change the SG outbound rule to point at another SG | A rule's destination type can't be changed after creation. Deleted it and added a new rule (`02`) |
| Session opened as root | Created `ssm-operator` and retested |

## In production

- Log every session's commands to CloudWatch Logs or S3 (history only shows *who* and *when*).
- Endpoints in every Availability Zone.
- Allow sessions by **tag** instead of instance ID.
- Use IAM Identity Center with MFA instead of IAM users.

## Key takeaways

- Session Manager flips the direction: the server dials out, so it needs **zero inbound rules**.
- IAM replaces SSH keys, and every session is tied to a named identity.
- Security groups are stateful: allowing outbound 443 is enough.
- An audit trail only helps if people sign in as themselves, not as root.

<details>
<summary><b>Request flow (click to expand)</b></summary>

| # | Who → who | Port | How it's authenticated |
| --- | --- | --- | --- |
| 1 | Agent → instance metadata (IMDSv2) | 80, link-local | IMDSv2 token; returns the role's temporary credentials |
| 2 | Agent → VPC DNS: `ssm.us-east-1.amazonaws.com`? | 53 | Private DNS answers `10.0.2.121` |
| 3 | Agent → `ssm` endpoint: register, check in | 443 | Signed with `ec2-ssm-role` credentials |
| 4 | Agent → `ssmmessages` endpoint (`10.0.2.7`): opens a long-lived channel | 443 | Signed with `ec2-ssm-role` credentials |
| 5 | Engineer → Systems Manager: `StartSession` | 443 | IAM user, checked against `ssm:StartSession` |
| 6 | Systems Manager → agent over the open channel; shell starts as `ssm-user` | existing 443 channel | Already authenticated |

</details>

<details>
<summary><b>Revision notes (click to expand)</b></summary>

**Why both VPC DNS settings are needed**
- *DNS resolution* = the VPC's DNS server (VPC base + 2, e.g. `10.0.0.2`) answers at all. Off → no names resolve.
- *DNS hostnames* = AWS can add private DNS names. Required for endpoint private DNS. Off → agent gets **public** SSM IPs, has no route to them, never registers.

**What each endpoint carries**
- `ssm`: registration and short calls (rarely visible in `ss`, they close quickly)
- `ssmmessages`: long-lived channels, including the live shell (shows as `ESTAB`)
- `ec2messages`: older command channel, still recommended

**If the node never appears in Fleet Manager, check in this order:** role attached → VPC DNS settings → private DNS on endpoints → endpoint SG allows 443 from instance SG → `ssmmessages` endpoint exists.

**Instance profile:** EC2 attaches a role through an *instance profile*. The console creates one with the same name automatically; with CLI/Terraform you create it yourself.

**Trust policy vs permissions policy:** trust = *who can wear the role* (here `ec2.amazonaws.com`); permissions = *what it can do*.

**Implicit vs explicit deny:** everything is denied unless allowed. *"No identity-based policy allows"* = implicit (permission never granted). *"With an explicit deny"* = someone wrote a `Deny`, which beats any Allow, even admin.

**Why `curl` timed out instead of being refused:** security groups drop disallowed traffic silently. Two layers blocked it: the SG (443 to endpoints only) and the route table (no internet route). That's defence in depth.

**Commands used on the instance**
```bash
whoami                                       # ssm-user = came in via Session Manager
getent hosts ssm.us-east-1.amazonaws.com     # name → private endpoint IP
getent hosts ssmmessages.us-east-1.amazonaws.com
sudo ss -tnp | grep 443                      # live TCP connections on 443, with process names
curl -m 5 https://www.google.com             # should time out: no internet path
```

**Teardown order:** terminate instance → delete endpoints (wait) → remove the instance SG's outbound rule (the two SGs reference each other) → delete endpoint SG → delete instance SG → delete `ssm-operator` and its policy. Keep `ec2-ssm-role`.

</details>
