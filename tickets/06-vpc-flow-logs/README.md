# Ticket 06: Show me the traffic hitting our servers

> **Request:** "Something strange is hitting our web server. Tell us who, on which ports, and whether anything got in."

**Status:** Done · **Date:** 2026-11-27 · **Region:** us-east-1

## Result

I turned on **VPC Flow Logs** for the lab VPC and exposed a web server with **only port 80 open**. After ~6 hours:

| Question | Answer |
| --- | --- |
| **Who?** | 1,000+ blocked attempts on an IP I never published. Biggest source: one network (`85.217.149.x`), **686 attempts from 55 IPs**. |
| **Which ports?** | Remote logins (SSH, Telnet, RDP, VNC), admin panels, MySQL, CCTV recorders, VoIP, pings. |
| **Got in?** | **No.** Outsiders only reached port 80, the website, which is open on purpose. |

## How it works

```
Internet ──▶ Internet gateway ──▶ flowlog-web (10.0.1.80)
                                   SG: allow HTTP 80 only
                                        │
             VPC Flow Logs records every connection (ACCEPT / REJECT)
                                        ▼
             CloudWatch Logs ──▶ Logs Insights queries
```

- **Flow logs** record *who connected, to which port, and whether it was allowed*. Not the contents.
- The Flow Logs service writes to CloudWatch using its own IAM role (`vpc-flow-logs-role`), limited to one log group and my account.
- No SSH on the server. Admin access is through Session Manager (Ticket 05).

## What I built

| Resource | Setting |
| --- | --- |
| Log group `/vpc/lab-flow-logs` | 1-week retention |
| Role `vpc-flow-logs-role` | Trusts `vpc-flow-logs.amazonaws.com` (my account only); can write to this log group only |
| Flow log | Whole VPC, **All** traffic, 1-minute interval |
| `flowlog-web` | Amazon Linux 2023 + Apache, public subnet, no key pair, SG allows **HTTP 80 only** |

→ `01-web-server-loads-http.png`

## Investigation

**1. Check the logs are accurate.** I made traffic I knew about: loaded the page, then tested port 22. The logs matched exactly: 80 ACCEPT, 22 REJECT. They also caught Chrome trying HTTPS first and a Windows NetBIOS lookup I didn't know I'd sent.
→ `02-test-netconnection-port22-blocked.png`, `03-my-own-traffic-verified.png`

**2. Avoid the trap.** My first "top sources" list was mostly **AWS and time servers replying to my own server** (source ports 443 and 123), not attackers. Filtering on `action = "REJECT"` shows only uninvited traffic.
→ `04-replies-vs-strangers.png`

**3. Who is attacking?** Grouping by network showed scanners spread across many IPs, so blocking single IPs is pointless. A default-deny security group stops them all.
→ `05-blocked-by-network.png`

```
filter dstAddr = "10.0.1.80" and action = "REJECT"
| parse srcAddr /(?<network>\d+\.\d+\.\d+)\.\d+/
| stats count(*) as attempts, count_distinct(srcAddr) as ips by network
| sort attempts desc
```

**4. Which ports?** Mostly remote-login ports and admin panels. All blocked.
→ `06-blocked-ports.png`

**5. Did anything get in?** Only port 80 accepted connections from outside.
→ `07-accepted-port80-only.png`

**6. What did website visitors want?** The Apache log showed a SQL injection probe, router-login hunting, config-file hunting, and a research scanner naming itself (Palo Alto Networks). The site is one static page, so there was nothing to exploit, even where the response was `200`.
→ `08-apache-access-log.png`

## Problems I hit

| Problem | Fix |
| --- | --- |
| IAM had no "VPC Flow Logs" use case | Wrote a **custom trust policy** |
| Logs Insights errors | I'd pasted the query into the log-group box, not the editor |
| Query returned 0 results | Time range was 1 hour; my tests were 5 hours earlier. Changed to 12h |
| Top "attackers" were AWS replies | Filtered on `REJECT` |

## In production

- Flow logs on every VPC, stored in a **separate logging account** so attackers can't delete them.
- Add the `flow-direction` field to tell incoming from replies instantly.
- Ship web server logs off the server (Ticket 07).
- GuardDuty to flag known-bad IPs automatically (Ticket 13), WAF to block web probes (Ticket 11).

## Key takeaways

- Any public IP gets scanned within hours. Default-deny isn't optional.
- Flow logs show **who and which port**. Web server logs show **what they asked for**.
- "Incoming" doesn't always mean "attack": check the source port.
- Group attackers by network, not single IPs.

<details>
<summary><b>Request flow (click to expand)</b></summary>

| # | Who → who | Port | How it's decided / authenticated |
| --- | --- | --- | --- |
| 1 | Visitor → `13.219.251.155` | any | Internet gateway translates to private IP `10.0.1.80` |
| 2 | Packet reaches the server's ENI | any | Security group: TCP 80 allowed, everything else silently dropped (REJECT) |
| 3 | Server → AWS / NTP (SSM agent 443, clock sync 123, `dnf` 443) | 443 / 123 | Outbound allowed; replies return automatically (stateful) and log as ACCEPT |
| 4 | Flow Logs service → CloudWatch Logs | n/a | Assumes `vpc-flow-logs-role` (trust: `vpc-flow-logs.amazonaws.com`, condition `aws:SourceAccount`) |
| 5 | Me → Logs Insights | 443 | My IAM identity |

NAT happens at both ends: my home router turns `192.168.1.149` into my public IP, and the internet gateway turns the public server IP into `10.0.1.80`. So flow logs show the **public source** and the **private destination**.

</details>

<details>
<summary><b>Revision notes (click to expand)</b></summary>

**Reading a flow log record** (default format)

| Field | Meaning |
| --- | --- |
| `interface-id` | Which ENI saw it. One log stream per ENI. |
| `srcaddr → dstaddr` | From → to |
| `srcport → dstport` | Sender's random ephemeral port → the port they targeted |
| `protocol` | A protocol **number**, not a port: 6 = TCP, 17 = UDP, 1 = ICMP (ICMP shows port 0) |
| `action` | ACCEPT or REJECT (SG or NACL decision) |

**Setup notes**
- Filter **All**, not Reject-only, or you can't compare allowed vs blocked.
- 1-minute aggregation shows records sooner. Expect 5 to 10 minutes' delay at first.
- Set retention, or logs pile up and cost money.
- The flow log covers the whole VPC, so other resources' ENIs (here, the Ticket 01 EFS mount targets) get their own streams.

**Telling replies from attacks**
- Replies to the server's own calls arrive *from* port 443 or 123 *to* a high port, and are ACCEPT.
- Uninvited traffic is REJECT, or ACCEPT to a **low** port (a service).

**Logs Insights tips**
- The top box (query scope) picks the **log group**. The editor below holds the **query**.
- The `START=` value on the `SOURCE` line sets the time window.

**All queries used**
```
# 1. My own traffic (checks the logs are accurate)
filter srcAddr = "<MY_HOME_IP>" | sort @timestamp desc

# 2. Who sent traffic in, and from which port (reveals replies)
filter dstAddr = "10.0.1.80" | stats count(*) as records by srcAddr, srcPort, action | sort records desc

# 3. Blocked traffic by network
filter dstAddr = "10.0.1.80" and action = "REJECT"
| parse srcAddr /(?<network>\d+\.\d+\.\d+)\.\d+/
| stats count(*) as attempts, count_distinct(srcAddr) as ips by network | sort attempts desc

# 4. Ports targeted
filter dstAddr = "10.0.1.80" and action = "REJECT" | stats count(*) as attempts by dstPort, protocol | sort attempts desc

# 5. Anything uninvited that got in (connections to a service port)
filter dstAddr = "10.0.1.80" and action = "ACCEPT" and dstPort < 1024 | stats count(*) by dstPort, srcAddr
```

**Ports that came up**

| Port | Service |
| --- | --- |
| 22 / 23 | SSH / Telnet (remote login) |
| 3389 / 5900 | RDP / VNC (remote desktop) |
| 443, 8080, 8443, 3000, 8000, 10000 | Web servers and admin panels |
| 3306 | MySQL |
| 34567 | CCTV recorders (DVRs) |
| 5060 (UDP) | SIP, internet phone systems |
| 123 (UDP) | NTP, time sync |
| 137 (UDP) | Windows NetBIOS |

**Reading the Apache log:** `IP - - [time] "request" status bytes "referrer" "user-agent"`. Status: `200` OK, `304` cached, `400` bad request, `404` not found, `408` connected but sent nothing. A `200` doesn't mean an attack worked: always ask whether the response did what the attacker wanted.

**Teardown:** delete the flow log → terminate `flowlog-web` → delete `flowlog-web-sg` → delete the log group when done. `vpc-flow-logs-role` is free to keep.

</details>
