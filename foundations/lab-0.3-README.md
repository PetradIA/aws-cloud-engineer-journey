# Lab 0.3 — AWS CLI Fluency

**Roadmap phase:** Phase 0 — Workbench Setup
**Author:** Taiwo

---

## Overview

This lab repeats Lab 0.1's work — checking and controlling an EC2 instance — but entirely through the AWS CLI instead of the console. The goal was to move from clicking through the console to querying and controlling infrastructure with commands, since the console doesn't scale and can't be scripted or automated later in the roadmap.

---

## What I built

- Authenticated the AWS CLI on the EC2 instance using a scoped IAM access key via `aws configure` (after `aws login`'s browser-based short-lived credentials failed to persist for CLI use)
- Verified CLI identity with `aws sts get-caller-identity`
- Queried and filtered EC2 instance data using `describe-instances` with `--query` (JMESPath) and different `--output` formats (`table`, `text`)
- Stopped and started the daily-driver EC2 instance (`i-0cd08c91f9d225cae`) via CLI and console, confirming state changes at each step

---

## Key concepts learned

- **`aws login` vs. `aws configure`** — `aws login` is meant to bridge browser-based console sessions to the CLI using short-lived credentials, but it didn't persist locally in this setup. A scoped IAM access key configured via `aws configure` was the reliable fallback — a normal, valid way to authenticate CLI access.
- **`--query` and JMESPath** — AWS CLI commands return large raw JSON by default; `--query` filters that down to just the fields needed, using JMESPath syntax (e.g., `Reservations[*].Instances[*].State.Name` walks into nested arrays/objects to pull out one field).
- **`--output` formats** — `table` is best for scanning multiple results at a glance; `text` is better for a single, specific value (like checking one instance's state).
- **CLI commands follow a consistent shape**: `program → service → action → flags`. Each `--flag` is a name-value pair — the CLI equivalent of filling out a form field.
- **A stopped instance can't run its own CLI commands** — since CLI access lived on the instance itself, starting it back up had to happen from the console, not from a terminal session that was already disconnected.

---

## What broke, and how I fixed it

`aws sts get-caller-identity` initially failed with "Unable to locate credentials," even after running `aws login` and seeing what looked like a successful browser authentication. Checking `aws configure list` confirmed the CLI had no access key, secret key, or profile stored at all — `aws login` never persisted anything locally for the CLI to use. The fix was creating a scoped IAM access key in the console and running `aws configure` directly with it, which authenticated cleanly.

---

## Full notes

📄 Full command reference: [`cheatsheet/aws-cli.md`](../../cheatsheet/aws-cli.md)

---

## Next up

Phase 0 mini project — documenting the full workbench setup (EC2 + Git + CLI) as an onboarding guide.


