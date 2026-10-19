# check_process role

Ansible 2.9 compatible, read-only process check. Existing before.yml and after.yml call this role.

Configure `process_by_group` in Tower inventory variables or group_vars:

```yaml
process_by_group:
  dev_print:
    - {name: netman, match: exact}
    - {name: mailman, match: exact}
  uat_jmdc:
    - {name: ora_pmon_JMDC, match: exact}
    - {name: hulsndd, match: exact}
```

These are **examples**, not verified mappings. Confirm processes for all ten groups before production.

Use `host_processes` in host_vars for exceptions:

```yaml
host_processes:
  - {name: '/apps/SVFJP/svfjpd/lib/svf.jar', match: cmdline}
```

`match: exact` uses pgrep -x; `match: cmdline` uses pgrep -f (substring matching may produce false positives). Existing `common_processes` and `extra_processes` string lists remain supported. All matching groups are combined; empty check lists fail rather than silently passing. Tower Limit selects hosts.

Validate with Ansible 2.9 syntax-check and real Tower inventory on test machines. No production execution or syntax validation has been performed.
