# Ansible Dev Activity Audit — Ansible Vault

## Problem Statement

We have a shared development server where multiple developers keep their project files under `/opt/dev/projects/`. There was no easy way to tell who is actively working and who hasn't touched their code in weeks. Checking this manually by SSH-ing into the server every week and running `find` commands was time-consuming and inconsistent.

This playbook automates that check.

---

## About the Playbook

This is an Ansible playbook that connects to the dev server, checks file activity in each developer's directory, and produces a report. Credentials are managed using **Ansible Vault** — the sudo password is stored in an encrypted file in this repo and decrypted at runtime using a vault password you keep locally.

If you want to use AWS Secrets Manager instead of Vault, use the `main` branch.

---

## What It Does

1. Connects to the dev server over SSH using a dedicated key
2. Checks that Python 3 is available and the OS is CentOS 7
3. Finds all developer directories under `/opt/dev/projects/`
4. For each developer, counts how many files were modified in the last 7 days
5. Finds the most recently modified file per developer
6. Writes a report to `/var/log/dev_activity_report.txt` on the server
7. Pulls a copy of the report to your machine under `reports/`

By default it only checks `blob` and `alice`. You can change that or scan everyone — see the run section.

---

## How to Run

**Before the first run — one time setup:**

```bash
# Generate SSH key and copy to server
ssh-keygen -t ed25519 -f ~/.ssh/ansible_id_ed25519 -C "ansible-audit"
ssh-copy-id -i ~/.ssh/ansible_id_ed25519.pub ansible@<server-ip>

# Create vault password file
echo "your-vault-password" > ~/.vault_pass
chmod 600 ~/.vault_pass

# Edit vault file with real sudo password then encrypt it
ansible-vault edit group_vars/dev_servers/vault.yml --vault-password-file ~/.vault_pass
ansible-vault encrypt group_vars/dev_servers/vault.yml --vault-password-file ~/.vault_pass

# Update the server IP
# Open host_vars/dev-server-01.yml and set ansible_host to the real IP
```

**Normal run:**

```bash
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

**If you prefer typing the password instead:**

```bash
ansible-playbook site.yml --ask-vault-pass
```

**Change the inactivity threshold (default is 7 days):**

```bash
ansible-playbook site.yml --vault-password-file ~/.vault_pass -e "audit_activity_days=14"
```

**Scan all developers instead of just blob and alice:**

```bash
ansible-playbook site.yml --vault-password-file ~/.vault_pass -e "audit_target_users=[]"
```

**Target staging instead of production:**

```bash
ansible-playbook site.yml -i inventories/staging/hosts --vault-password-file ~/.vault_pass
```

---

## How to Validate

**Dry run first — no changes will be made:**

```bash
ansible-playbook site.yml --vault-password-file ~/.vault_pass --check --diff
```

This shows what the playbook would do without actually doing it. Good habit before running against production.

**Run only specific phases to test:**

```bash
# Just check connectivity and preflight assertions
ansible-playbook site.yml --vault-password-file ~/.vault_pass --tags "preflight"

# Run discovery only — check it finds the right directories
ansible-playbook site.yml --vault-password-file ~/.vault_pass --tags "discover"

# Full run except the report write
ansible-playbook site.yml --vault-password-file ~/.vault_pass --tags "preflight,discover,collect"
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
ansible-playbook site.yml --vault-password-file ~/.vault_pass -v
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
| Credentials | Ansible Vault — encrypted file in `group_vars/dev_servers/vault.yml` |
| Default inventory | `inventories/production/hosts` |

The vault file (`group_vars/dev_servers/vault.yml`) must be encrypted before pushing anywhere. Never commit it in plaintext. Keep your `~/.vault_pass` file out of the repo — it's already in `.gitignore`.
