# Existing check_process.sh output — sanitized reference

This is an **illustrative redacted transcription** of the supplied Tower output, not a live process listing. All host names, NAS sources, PID values and internal identifiers are omitted.

```text
NAS mount check:
  <NAS_SOURCE> 5.0G 454M 4.6G 9% /data/<MOUNT_PATH>

HULFT:
  <USER> <PID> ... ./hulrcvd -l /logs/hulft/hulcon.log
  <USER> <PID> ... ./hulsndd -l /logs/hulft/hulcon.log
  <USER> <PID> ... ./hulobsd
  <USER> <PID> ... ./huledd

TWS (FTA):
  <USER> <PID> ... /opt/TWA86/TWS/bin/netman
  <USER> <PID> ... /opt/TWA86/TWS/bin/batchman
  <USER> <PID> ... /opt/TWA86/TWS/bin/mailman
  <USER> <PID> ... /opt/TWA86/TWS/bin/monman
  <USER> <PID> ... /opt/TWA86/TWS/bin/writer
  <USER> <PID> ... /opt/TWA86/TWS/bin/jobman

Splunk:
  <USER> <PID> ... splunkd -p 8089 start
  <USER> <PID> ... splunkd -p 8089 start [process-runner]
```

## Design and caveats

- The existing script is retained as the source of process listings. Its exit code is checked.
- Configure expected process fragments by inventory group, with optional host additions.
- Empty configurations fail closed. Raw script output is hidden from Tower logs to avoid disclosing host identifiers.
- This checks **text output**, not live process health or reliable NAS mount state. The script can print stale/irrelevant lines. For production, prefer direct `pgrep` and `findmnt`/OS-specific mount verification.
- Mount checks here only assert a listed mount path, not that the mount is healthy or writable.
- Ansible 2.9 compatibility and Tower behavior must be tested; no runtime tests have been performed.

Example Tower/group vars (placeholder group, no server IDs):

```yaml
check_process_by_group:
  example_print:
    - /opt/TWA86/TWS/bin/netman
    - /opt/TWA86/TWS/bin/mailman
    - /opt/TWA86/TWS/bin/batchman
    - /opt/TWA86/TWS/bin/jobman
    - /opt/TWA86/TWS/bin/monman
    - /opt/TWA86/TWS/bin/writer
    - splunkd
check_process_required_mounts: []
```

The real per-group process requirements and NAS mount paths must be confirmed before enabling.
