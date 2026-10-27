# Reuse one Tomcat stop role for API / COMMON / FRONT

Compatible with Ansible 2.9 syntax. These are **examples** based on the
three role snippets; the actual Tomcat script is not stored in this repository.

## Layout

- `roles/tomcat_stop/tasks/main.yml` - shared Shell command and debug output
- `tomcat_stop.yml` - launch **one** requested component via `tomcat_target`
- `examples/tomcat_stop_three_calls.yml` - show three explicit `include_role` calls

## Best for separate Tower jobs

Create three Job Templates using the **same** playbook `tomcat_stop.yml`,
but set different Extra Variables/Survey values:

| Job Template | tomcat_target |
| --- | --- |
| Stop Tomcat API | API |
| Stop Tomcat COMMON | COMMON |
| Stop Tomcat FRONT | FRONT |

Example Extra Variables:

```yaml
tomcat_target: API
```

The playbook has no default, so a missing target fails validation instead
of stopping the wrong component.

## How the shared role works

It runs the same original command with a configurable final argument:

```sh
./tools/tomcat/tomcat_stop.sh API
./tools/tomcat/tomcat_stop.sh COMMON
./tools/tomcat/tomcat_stop.sh FRONT
```

`tomcat_stop_result.stdout_lines` is printed for each invocation.

To call it from an existing playbook:

```yaml
- name: Stop Tomcat API
  include_role:
    name: tomcat_stop
  vars:
    tomcat_target: API
```

The three-call example runs all three sequentially in one job. If a command
fails, Ansible normally skips the subsequent tasks for that host. If you
need genuinely independent success/failure status and execution, use the
three separate Tower Job Templates.

## Important

The original relative path `./tools/tomcat/tomcat_stop.sh` is preserved.
Ansible runs Shell tasks on the managed host: make sure the script exists
relative to the remote working directory (or use `args: chdir:` to select
the actual directory). No live stop command was executed by this example.
