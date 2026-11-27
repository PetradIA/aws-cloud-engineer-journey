# AWS Cloud Security Engineering Journey

Hands-on AWS labs on my way to becoming a **Cloud Security Engineer**. Every lab is written up like a real work ticket: the request, what I built, how I tested it, what broke, and what I'd do in production.

**Author:** Taiwo · Electrical/Electronics Engineering graduate · GitHub: [petradia](https://github.com/petradia)

## How this repo is organised

| Folder | What's inside |
| --- | --- |
| [`tickets/`](tickets/) | The main work. One folder per ticket, each with a README and screenshot evidence. **Start here.** |
| [`foundations/`](foundations/) | My first labs: setting up the AWS account, IAM user, EC2 Linux workbench, Git, and the AWS CLI. |
| [`cheatsheet/`](cheatsheet/) | AWS CLI commands I use, explained, updated as I learn new ones. |

## How I work

Each ticket is treated like a task on a real cloud team:

1. **Request:** a problem in plain words, the way a team lead or auditor would raise it.
2. **Build:** solve it in my own AWS account, with security built in from the start.
3. **Break it on purpose:** test that the wrong people, servers, or connections are actually blocked.
4. **Write it up:** what I built, what went wrong, how I fixed it, and what I'd change in production.

## Tickets

| # | Ticket | Skills | Status |
| --- | --- | --- | --- |
| 01 | [Shared storage for two app servers, with access control](tickets/01-efs-shared-storage/) | EFS, IAM roles, resource policies, TLS, Linux permissions, VPC endpoints | Done |
| 02 | "Our website went down when one server crashed. Make it survive that." | ALB, Auto Scaling, multi-AZ, security group chaining | Done (write-up coming) |
| 03 | "The app needs a database, and the auditor says no passwords in code." | RDS Multi-AZ, Secrets Manager | Done (write-up coming) |
| 04 | [Static site from a private S3 bucket, served through CloudFront](tickets/04-s3-cloudfront-private-site/) | S3, CloudFront, Origin Access Control, bucket policies | Done |
| 05 | [Shell access to a private server with no SSH](tickets/05-ssm-session-manager/) | SSM Session Manager, VPC interface endpoints, private DNS, least-privilege IAM, audit trail | Done |
| 06 | [Show me the traffic hitting our servers](tickets/06-vpc-flow-logs/) | VPC Flow Logs, CloudWatch Logs Insights, Apache access logs, traffic investigation | Done |
| 07 | "Nobody noticed the outage until a customer called." | CloudWatch agent, alarms, SNS | Next |
| 08 | "Someone changed a security group and something broke. Who did it?" | CloudTrail, EventBridge, SNS | Planned |
| 09 | "A contractor needs to start and stop dev servers only, for 2 weeks." | STS AssumeRole, trust policies, temporary credentials | Planned |
| 10 | "Customer data must be encrypted and we must control the keys." | KMS customer-managed keys, S3 SSE-KMS, CloudFront | Planned |
| 11 | "Bots are hammering the website." | AWS WAF, rate-based rules, CloudFront | Planned |
| 12 | "Put the whole app behind one secure front door." | CloudFront, ALB, EC2, RDS (full 3-tier request path) | Planned |
| 13 | "An access key leaked on GitHub. What now?" | GuardDuty, CloudTrail, incident response | Planned |
| 14 | "Security audit next week. Find what's wrong." | Security Hub, AWS Config, remediation | Planned |
| 15 | "Prove our defences actually work." | Red team / blue team: Kali vs Ubuntu in an isolated VPC, detection | Planned |

Each ticket adds one main new idea and builds on what earlier tickets set up. Every write-up includes a **request flow** section tracing who talks to whom, on which port, and how each call is authenticated.

## Where this is heading

After the tickets above: deeper IAM, logging and detection, data protection, and attack-to-defend labs, then the AWS Security Specialty exam, then Terraform, Python automation, and secure CI/CD. Larger portfolio projects will get their own repos and be linked here.
