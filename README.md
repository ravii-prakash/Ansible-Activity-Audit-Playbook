# Ansible Dev Activity Audit

## Problem Statement

We have a shared development server where multiple developers keep their project files under `/opt/dev/projects/`. There was no easy way to tell who is actively working and who hasn't touched their code in weeks. Checking this manually by SSH-ing into the server every week and running `find` commands was time-consuming and inconsistent.

This playbook automates that check.

---

## About the Playbook

This is an Ansible playbook that connects to the dev server, checks file activity in each developer's directory, and produces a report. Credentials are managed using Credential Manager (like AWS Secrets Manager or Ansible Vault) — the SSH private key and sudo password are stored in a secret and fetched at runtime. Nothing sensitive lives in the repo.

---

## What It Does

1. Fetches the SSH private key and sudo password from Credential Manager (like AWS Secrets Manager or Ansible Vault).
2. Writes the SSH key to a temp file (`/tmp/`) with `0600` permissions
3. Connects to the dev server using that key
4. Checks that Python 3 is available and the OS is CentOS 7
5. Finds all developer directories under `/opt/dev/projects/`
6. For each developer, counts how many files were modified in the last 7 days
7. Finds the most recently modified file per developer
8. Writes a report to `/var/log/dev_activity_report.txt` on the server
9. Pulls a copy of the report to your machine under `reports/`
10. Shreds the temp SSH key file so nothing is left on disk

By default it only checks `blob` and `alice`. You can change that or scan everyone — see the run section.

---

## How to Run

**Before the first run — one time setup:**

```bash
# Install the required collection
ansible-galaxy collection install -r requirements.yml # Used the AWS Secret Manger Hence need to install the module.
# Update host_vars/dev-server-01.yml with the real server IP
```

**Normal run:**

```bash
ansible-playbook site.yml
```

**Change the inactivity threshold (default is 7 days):**

```bash
ansible-playbook site.yml -e "audit_activity_days=14"
```

**Scan all developers instead of just blob and alice:**

```bash
ansible-playbook site.yml -e "audit_target_users=[]"
```

**Target staging instead of production:**

```bash
ansible-playbook site.yml -i inventories/staging/hosts
```

---

## How to Validate

**Dry run first — no changes will be made:**

```bash
ansible-playbook site.yml --check --diff
```

**Run only specific phases to test:**

```bash
# Just check connectivity and preflight assertions
ansible-playbook site.yml --tags "preflight"

# Run discovery only — check it finds the right directories
ansible-playbook site.yml --tags "discover"

# Full run except the report write
ansible-playbook site.yml --tags "preflight,discover,collect"
```

**Check the report was created on the server:**

```bash
ssh ansible@<server-ip> "sudo cat /var/log/dev_activity_report.txt"
```

**Check the local copy was fetched:**

```bash
ls reports/
cat reports/dev-server-01_dev_activity_*.txt
```

**Check the events log (updated every time the report changes):**

```bash
ssh ansible@<server-ip> "sudo cat /var/log/dev_audit_events.log"
```

**Run with verbose output to see every task:**

```bash
ansible-playbook site.yml -v
```

---

## Summary

| What | Detail |
|---|---|
| Target server OS | CentOS 7 |
| Default users audited | blob, alice |
| Inactivity threshold | 7 days (configurable) |
| Report on server | `/var/log/dev_activity_report.txt` |
| Report on your machine | `reports/<hostname>_dev_activity_<date>.txt` |
| Credentials | AWS Secrets Manager — nothing sensitive in the repo |
| Default inventory | `inventories/production/hosts` |
| Secret name | `dev-audit/ansible-credentials` (configurable in `group_vars/all/vars.yml`) |
| AWS region | `ap-southeast-2` (configurable) |
