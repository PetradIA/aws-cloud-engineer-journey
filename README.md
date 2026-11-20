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
| 02 | "Our website went down when one server crashed. Make it survive that." | ALB, Auto Scaling, multi-AZ, security group chaining | Next |
| 03 | "The app needs a database, and the auditor says no passwords in code." | RDS Multi-AZ, Secrets Manager | Planned |
| 04 | "Marketing needs a static site, and security says the bucket can't be public." | S3, CloudFront, Origin Access Control | Planned |
| 05 | "We got a surprise bill and nobody noticed the outage." | CloudWatch, SNS, billing alarms | Planned |
| 06 | "Someone changed a security group and something broke. Who did it?" | CloudTrail, AWS Config | Planned |
| 07 | "A contractor needs to start and stop dev servers only, for 2 weeks." | IAM permission boundaries, temporary credentials | Planned |
| 08 | "Customer data must be encrypted and we must control the keys." | S3, KMS customer-managed keys | Planned |
| 09 | "An access key leaked on GitHub. What now?" | GuardDuty, CloudTrail, incident response | Planned |
| 10 | "Security audit next week. Find what's wrong." | Security Hub, remediation | Planned |

## Where this is heading

After the tickets above: deeper IAM, logging and detection, data protection, and attack-to-defend labs, then the AWS Security Specialty exam, then Terraform, Python automation, and secure CI/CD. Larger portfolio projects will get their own repos and be linked here.
