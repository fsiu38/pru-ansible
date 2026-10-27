# Unified Tomcat start role (Ansible 2.9)

This branch adds one `tomcat_start` role to replace the three similar role
snippets shown for `tomcat_start_api`, `tomcat_start_common` and
`tomcat_start_front`. It does not modify the live company playbooks.

## Behavior

1. Stop the selected Tomcat component.
2. Wait **10 seconds**.
3. Start that component.
4. Print the start command's `stdout_lines`.

The original `FRONT` role also showed its stop output, so this combined role
prints preliminary-stop output for `FRONT` only.

The photographed API/COMMON shell tasks separated their stop, sleep and start
commands with `;`, which meant that a failed stop did not prevent an attempted
start. To preserve that behavior, this role tolerates a non-zero preliminary
stop return code for API/COMMON. FRONT's stop task still fails fast if nonzero.
The start task itself fails normally if its shell command returns nonzero.

## Three independent host groups (same playbook)

See `examples/tomcat_start_three_groups.yml`:

```yaml
- hosts: tomcat_front
  gather_facts: false
  roles:
    - role: tomcat_start
      tomcat_target: FRONT

- hosts: tomcat_common
  gather_facts: false
  roles:
    - role: tomcat_start
      tomcat_target: COMMON

- hosts: tomcat_api
  gather_facts: false
  roles:
    - role: tomcat_start
      tomcat_target: API
```

Each play selects its own inventory hosts. These plays execute sequentially.
For truly independent Tower Job results, use three distinct Job Templates.

## Tower Job Template approach

The shared `tomcat_start.yml` uses `hosts: all`, so configure
**Limit** per Job Template and pass `tomcat_target` using Survey/Extra Variables:

| Tower Job Template | Limit | Extra vars |
| --- | --- | --- |
| Start FRONT | `tomcat_front` | `tomcat_target: FRONT` |
| Start COMMON | `tomcat_common` | `tomcat_target: COMMON` |
| Start API | `tomcat_api` | `tomcat_target: API` |

A missing or unsupported `tomcat_target` fails validation.

## Script paths and scope

The three photographed **start** roles use absolute script paths
`/tools/tomcat/tomcat_stop.sh` and `/tools/tomcat/tomcat_start.sh`, which
this role preserves.

**Notice:** The separate `tomcat_stop` role in this demo branch uses the
relative script path `./tools/tomcat/tomcat_stop.sh`, inherited from an earlier
photo. Verify both path conventions on each managed host before deployment.

This is a code example only. No target-server commands were executed.
