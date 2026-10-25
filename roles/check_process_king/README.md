# check_process_king

Read-only per-host process checks (Ansible 2.9 compatible).

Set `before_check_process` in inventory variables:

```yaml
before_check_process:
  app01:
    - process_a
    - "xpopserver stat"
```

The quoted entry with a space is **one list item**, passed as **one shell argument**
because the task uses `{{ item | quote }}`. The script must accept that
entire string as its first argument, rather than expecting two arguments.

Invoke with `roles: [check_process_king]`. Unlisted hosts are skipped.
Verify the script accepts an argument and is read-only before use.
`check_mode: false` intentionally runs it even under Ansible check mode.

Results are registered as `before_check_process_result`; failures are recorded
but do not fail the play (`failed_when: false`). Review `rc` and `stdout`.
This is a testing role, not a production gate.

No internal server names are committed because this repository is public.
