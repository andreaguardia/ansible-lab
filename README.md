Ansible playbooks that manage three MikroTik routers (RouterOS 7) and the Ubuntu automation server of my home lab. The playbooks are **idempotent** and the router password is encrypted with **Ansible Vault**.

This is Project 5 of my networking lab series, after
[mikrotik-ospf-lab](https://github.com/andreaguardia/mikrotik-ospf-lab) and
[netmiko-automation](https://github.com/andreaguardia/netmiko-automation).

## Lab

| Host | Type | Management IP | Connection |
|---|---|---|---|
| R1, R2, R3 | MikroTik CHR (RouterOS 7) | 192.168.233.11 / .12 / .13 | SSH (`network_cli`) |
| ubuntu | Ubuntu VM (Ansible control node) | 192.168.233.128 | local |

The routers run OSPF in a triangle (see `mikrotik-ospf-lab`). Ansible reaches them over their management interface (`ether1`).

## Files

| File | Purpose |
|---|---|
| `ansible.cfg` | Default inventory, no host-key prompt, path to the vault password file |
| `inventory.yml` | Two groups: `routers` (R1–R3) and `linux` (the Ubuntu VM) |
| `group_vars/routers/vault.yml` | Router password, encrypted with Ansible Vault (**not uploaded**) |
| `group_vars/routers/vault.yml.example` | Example of the vault file, without the real password |
| `backup.yml` | Exports the configuration of every router to `backups/` with a timestamp |
| `config.yml` | Applies the same settings to all routers |
| `linux.yml` | Configures the Ubuntu server (see below) |
| `requirements.txt` | Python packages for the virtual environment |

### What `linux.yml` does

- Fixes clock drift after the VM is suspended: sets `makestep 1 -1` in chrony and restarts chrony through a **handler**, only when the file changed
- Installs `htop`, `tree` and `snmp` with apt (`cache_valid_time: 3600`)
- Creates a read-only user, `nocviewer`
- Schedules the daily router backup with cron at 03:00
- Prints system facts (OS, IP, memory) collected by Ansible

## Concepts practiced

- **Inventory and groups**: one file describes every device and how to connect to it
- **Ad-hoc commands** vs. **playbooks**
- **Idempotency**: running a playbook a second time reports `changed=0`, because Ansible only changes what is not already in the desired state
- **Handlers** (`notify`): restart a service only when its configuration changed
- **register / debug**: save a task's output and print it
- **delegate_to / run_once**: run a task on the control node only once, for example to create the backup folder
- **Facts**: information Ansible collects automatically from each host
- **Ansible Vault**: encrypted secrets that can live next to the code

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install community.routeros

# Create the vault with the router password
ansible-vault create group_vars/routers/vault.yml
#   vault_net_password: "your-password"

# Save the vault password so ansible.cfg can use it (local only, never committed)
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass
```

## Usage

```bash
# Ad-hoc test: ask every router for its name
ansible routers -m community.routeros.command -a "commands='/system identity print'"

# Playbooks
ansible-playbook backup.yml
ansible-playbook config.yml
ansible-playbook linux.yml
```

## Security

- The router password is encrypted with **Ansible Vault**; the vault file and `.vault_pass` are listed in `.gitignore`
- A personal user (`netadmin`) is used instead of `admin`, and only SSH is enabled on the routers
- Backups stay local (`backups/` is ignored by Git) because router configurations can contain sensitive data

## Problems I solved

| Problem | Cause | Fix |
|---|---|---|
| Vault error "expected dicts" | Vault content was a plain string | Wrote it as a key: `vault_net_password: "..."` |
| `become` timed out on Ubuntu | Ubuntu uses `sudo-rs` | `ansible_become_exe: sudo.ws` |
| apt: "Release file is not valid yet" | VM clock behind after suspend | chrony `makestep 1 -1` |
| chrony restarted on every run (not idempotent) | Restart was a normal task | Moved it to a handler |
| Deprecation warning for facts | Old variable names | Used `ansible_facts[...]` |
| YAML errors in `linux.yml` | Wrong indentation | Fixed the spaces |