# Ansible 2.9 Demo — uname and per-host process checks

This demo targets **Linux** hosts. All IP addresses (`192.0.2.x`) are **documentation placeholders**; replace them with real host addresses and configure SSH credentials before running.

## Directory layout

```text
ansible-demo/
├── ansible.cfg
├── site.yml
├── inventory/
│   ├── hosts.yml
│   ├── group_vars/webservers.yml
│   └── host_vars/{web01,web02,db01,app01}.yml
└── roles/check_process/
    ├── defaults/main.yml
    └── tasks/main.yml
```

## Run

```bash
cd ansible-demo
ansible-inventory --graph
ansible all -m ping
ansible-playbook site.yml
# Run for one host:
ansible-playbook site.yml --limit web01
```

## Design

- `site.yml`: runs `uname -a` and then includes the process-check Role.
- `inventory/group_vars/`: common processes for a group.
- `inventory/host_vars/`: extra processes on individual hosts.
- `roles/check_process/tasks/main.yml`: checks every expected process with `pgrep -x` and fails via `assert` if missing.
- `changed_when: false`: read-only checks are never reported as changes.

**Notes**: `pgrep -x` matches the executable/process name, not the full command line; Linux process names may be truncated to 15 bytes. For Java applications or processes requiring argument matching, use a more specific check. `pgrep` must be installed. A process check is not the same as a service-health check. Empty lists skip the process checks.

## Compatibility

Uses Ansible 2.9 module short names (`command`, `debug`, `include_role`, `set_fact`, `assert`). Check with `ansible-playbook --syntax-check site.yml` on your controller. This package was statically reviewed, not run against an Ansible 2.9 controller or live hosts. `pgrep` must exist on target Linux hosts.
