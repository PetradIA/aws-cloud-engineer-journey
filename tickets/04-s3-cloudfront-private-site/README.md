# Ticket 04: Static site from a private S3 bucket, served through CloudFront

> **Request:** "Marketing needs a static site, and security says the bucket can't be public."

**Status:** Done · **Date:** 2026-11-24 · **Region:** us-east-1

## Result

The Tee Tutors page is live at `https://dmuub36y1e6i7.cloudfront.net`. The files sit in a **private** S3 bucket (`tee-tutors-site`, Block Public Access on). Only CloudFront can read them:

- CloudFront **signs** every request to S3 using **Origin Access Control (OAC)**.
- The **bucket policy** trusts only CloudFront, only **this one distribution**, and only for reading files.

Opening the S3 URL directly returns `AccessDenied`. Visitors get HTTPS automatically, and the page is cached close to them.

## How it works

```
 User ──HTTPS──▶ CloudFront (dmuub36y1e6i7.cloudfront.net)
                 • HTTP → HTTPS redirect, default root object index.html
                 • cached at edge locations
                        │ request signed with OAC
                        ▼
                 S3 bucket tee-tutors-site (private)
                 • bucket policy: this distribution only, read only
                        ▲
 User ──direct S3 URL──X   403 AccessDenied
```

- **Origin** = where the files live (the bucket). **Distribution** = the CloudFront setup that serves them worldwide.
- The bucket stays fully private and still serves a public website, because CloudFront is the only thing allowed in.

## What I built

| Resource | Setting |
| --- | --- |
| S3 bucket `tee-tutors-site` | Block all public access ON, ACLs off, SSE-S3 encryption, **no** static website hosting |
| `index.html` | One page, stored as `text/html` ([`site/index.html`](site/index.html)) |
| CloudFront distribution | Origin = S3 **REST** endpoint, HTTP → HTTPS, default root object `index.html` |
| OAC | Created by the "Allow private S3 bucket access to CloudFront" option |
| Bucket policy | Written by CloudFront: `s3:GetObject` for this distribution only |
| WAF | Not enabled here; saved for Ticket 11 on this same distribution |

## Tests

| # | Test | Result | Evidence |
| --- | --- | --- | --- |
| 1 | S3 URL before CloudFront | `AccessDenied` | `01` |
| 2 | Create distribution with private S3 access | OAC created, bucket policy written | `02`, `03` |
| 3 | Default root object after creation | Was blank, fixed to `index.html` | `04` |
| 4 | Open the CloudFront URL | Loads over HTTPS | `05` |
| 5 | S3 URL after setup | Still `AccessDenied` | `06` |

Evidence numbers match the files in [`screenshots/`](screenshots/).

## Problems I hit

| Problem | Fix |
| --- | --- |
| File uploaded as `index.html.txt` | Windows hid the extension. Turned on "File name extensions", renamed, re-uploaded, checked type is `text/html` |
| CloudFront returned 403 at `/` | Default root object was blank, so `/` asked S3 for the bucket itself, which the policy doesn't allow. Set it to `index.html` |

## In production

- Custom domain with an ACM certificate (issued in us-east-1) and a Route 53 alias.
- AWS WAF with managed rules and a rate limit (Ticket 11).
- Security headers policy (HSTS, CSP, etc.) and minimum TLS 1.2.
- CloudFront and S3 access logs to a locked-down logging bucket.
- S3 versioning to roll back bad uploads; deploy through CI/CD with a cache invalidation.
- Build it in Terraform.

## Key takeaways

- A bucket can be completely private and still back a public website.
- The `AWS:SourceArn` condition ties the bucket's trust to **one** distribution (prevents the confused deputy problem).
- Use OAC (current), not OAI (legacy), with the S3 **REST** endpoint, not website hosting.
- A blank default root object is a common cause of CloudFront 403s.
- Always show file extensions on Windows.

<details>
<summary><b>Request flow (click to expand)</b></summary>

| # | Who → who | Port | How it's authenticated |
| --- | --- | --- | --- |
| 1 | Visitor → CloudFront edge | 443 (80 redirected) | None needed for visitors; CloudFront's `*.cloudfront.net` certificate gives HTTPS |
| 2 | Edge cache hit → page returned | — | Served from cache, S3 not contacted |
| 3 | Cache miss: CloudFront → S3 REST endpoint | 443 | Request signed (SigV4) by OAC as the CloudFront service |
| 4 | S3 checks the bucket policy | — | Principal `cloudfront.amazonaws.com` + `AWS:SourceArn` = this distribution → `GetObject` allowed |
| 5 | Visitor → S3 URL directly | 443 | Anonymous request, no Allow + Block Public Access → 403 |

</details>

<details>
<summary><b>Revision notes (click to expand)</b></summary>

**Bucket policy (written by CloudFront)**
```json
{
  "Version": "2008-10-17",
  "Id": "PolicyForCloudFrontPrivateContent",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::tee-tutors-site/*",
      "Condition": {
        "ArnLike": { "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>" }
      }
    }
  ]
}
```

| Part | Meaning |
| --- | --- |
| `Principal: cloudfront.amazonaws.com` | **Who:** the CloudFront service, not a user |
| `Action: s3:GetObject` | **What:** read files only. No list, upload, delete |
| `Resource: tee-tutors-site/*` | **Which:** objects **inside** the bucket (`/*`), not the bucket itself |
| `Condition: AWS:SourceArn` | **Only when** the request comes from my distribution |

**Confused deputy problem:** without the `SourceArn` condition, the bucket would trust *any* CloudFront distribution in *any* account. Someone could point their own distribution at my bucket and read it.

**Policy version:** CloudFront wrote `2008-10-17`. It works, but for my own policies I use `2012-10-17`, which supports policy variables like `${aws:username}`.

**Why `/` returned 403 without a default root object:** a request for `/` becomes a request for the bucket itself, which is not `s3:GetObject` on an object, so the policy doesn't allow it.

**OAC vs OAI:** OAC is the current method (supports SigV4, SSE-KMS, all regions). OAI is legacy. Both only work with the S3 **REST** endpoint, not the `s3-website` endpoint.

**If using SSE-KMS later:** the KMS key policy must also allow the CloudFront service principal, with the same `SourceArn` condition (Ticket 10).

</details>
