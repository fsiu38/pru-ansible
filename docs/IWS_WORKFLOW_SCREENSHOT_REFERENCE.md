# IWS Before / After Workflow — Screenshot Reference

> Reference only. Transcribed from three photos of `IWS_後作業.yml` on 2026-10-20.
> This is NOT a tested or executable playbook. No production server identifiers are included.
> The IWS *before* workflow is not yet provided in these photos.

## After workflow (visible sequence)

| Order | Play target | Operation | Role | Loop input |
|---|---|---|---|---|
| 1 | `fta_dev` | Start IWSFTA | `Workstation_control/start_iwsfta` | none |
| 2 | `mdm_dev` | FTA Link | `Workstation_control/fta_link` | `groups['fta_dev']` |
| 3 | `fta_dev` | IWSFTA Process Check | `Workstation_control/fta_process_check` | none |
| 4 | `mdm_dev` | FTA WS Limit Control (After) | `Workstation_control/fta_ws_limit_control_after` | `groups['fta_dev']` |
| 5 | `mdm_dev` | RMS TDLIST | `Workstation_control/after_fta_check` | `IWSFTA_Aftercheck_JOB` |

The first four operations are visible in screenshot 2; the fifth in screenshot 3.

### Representative structure (not verbatim)

```yaml
# Illustrative only; the role implementations are not included.
- hosts: fta_dev
  become: true
  become_method: sudo
  become_user: root
  gather_facts: no
  tasks:
    - block:
        - name: Start IWSFTA
          include_role:
            name: Workstation_control/start_iwsfta
      rescue:
        - name: Start IWSFTA failure
          debug:
            msg: "Start IWSFTA failed!"
        - meta: end_play

- hosts: mdm_dev
  become: false
  gather_facts: no
  tasks:
    - block:
        - name: FTA Link
          include_role:
            name: Workstation_control/fta_link
          vars:
            fta_target: "{{ item }}"
          loop: "{{ groups['fta_dev'] }}"
      rescue:
        - name: FTA Link failure
          debug:
            msg: "FTA Link failed!"
```

## Other visible code (screenshot 1)

- `mdm_dev`: `Workstation_control/fta_ws_limit_control_before` using `fta_target_ws: "{{ item }}"` and `groups['fta_dev']`.
- `mdm_dev`: `Workstation_control/fta_unlink` using `fta_target_ws: "{{ item }}"` and `groups['fta_dev']`.
- Commented-out `fta_dev` play: `Workstation_control/stop_iwsfta` (not active).

These appear to be **before-work operations shown within the photographed file**, but their placement relative to the after workflow should be verified against the complete original.

## Failure handling

- Each operation uses `block` and `rescue`.
- Screenshot 2 shows `debug` followed by `meta: end_play` for failures.
- Screenshot 1 shows `ansible.builtin.fail` in `rescue` for the WS-limit-before and unlink steps.
- Screenshot 3 shows `ansible.builtin.fail` and `meta: end_play` for RMS TDLIST.
- **Review needed:** `meta: end_play` stops the current play; it is not equivalent to stopping an entire multi-play workflow.

## Design questions for later

1. Which source file is authoritative for before vs after operations?
2. Does `IWSFTA_Aftercheck_JOB` contain job names or another collection?
3. Should failures abort the entire Tower workflow or only the current play?
4. Are the `fta_dev` and `mdm_dev` groups environment-specific?
5. What are the role-level commands, checks, and success conditions?

Do not treat this screenshot reference as proof of successful execution.
