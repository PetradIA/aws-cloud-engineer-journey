# AWS CLI Cheatsheet

Useful commands collected while working through this roadmap — not meant to be memorized, just a personal quick-reference.

---

## Identity & configuration

**Check which identity the CLI is currently authenticated as**
```bash
aws sts get-caller-identity
```

**See current CLI configuration (profile, keys, region)**
```bash
aws configure list
```

**List all configured profiles**
```bash
aws configure list-profiles
```

**Set up CLI credentials interactively**
```bash
aws configure
```

---

## EC2

**List all EC2 instances, showing just ID / state / type**
```bash
aws ec2 describe-instances --query "Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,Type:InstanceType}" --output table
```
- `--query` filters the huge default JSON response using JMESPath
- `Reservations[*].Instances[*]` walks into every reservation, then every instance inside it
- `.{ID:...,State:...,Type:...}` builds a custom object with just the fields you want
- `--output table` formats the result as a readable ASCII table instead of raw JSON

**Stop an instance**
```bash
aws ec2 stop-instances --instance-ids <instance-id>
```

**Start a stopped instance**
```bash
aws ec2 start-instances --instance-ids <instance-id>
```

---

## Notes

- `--query` uses JMESPath syntax — fiddly to memorize, fine to look up each time
- `--output table` / `--output json` / `--output text` control how results are displayed; table is easiest to read at a glance


