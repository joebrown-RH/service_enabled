# service_audit_native

A mirror of the `service_audit` role that collects the same evidence using only
native Ansible modules — no `command`, no `shell`. It publishes the identical
`sox_audit` contract, so the `audit_report` role renders either one unchanged.

Run it with `playbooks/sox_service_audit_native.yml`.

## What replaced what

| `service_audit` | `service_audit_native` |
|---|---|
| `systemctl list-unit-files --state=enabled` | `ansible.builtin.service_facts` |
| `systemctl show <unit> --property=…` | `ansible.builtin.find` + `ansible.builtin.slurp` on the unit file and its drop-ins |
| `systemctl show --property=DefaultStandard*` | `slurp` of `/etc/systemd/system.conf` |
| `systemd-analyze cat-config systemd/journald.conf` | `find` + `slurp` of `journald.conf` and `journald.conf.d/*.conf` |
| `grep -RhsE … /etc/rsyslog.conf /etc/rsyslog.d/` | `find` + `slurp` + `regex_findall` |
| `auditctl -l \| grep -c` | `find` + `slurp` of `/etc/audit/rules.d/*.rules` |
| `stat` | unchanged — already native |

Unit files are resolved through systemd's own precedence order
(`/etc` → generators → `/run` → `/usr/lib`), and `*.service.d/*.conf` drop-ins
are layered on top with last-match-wins parsing, so an admin override is
honoured the same way systemd would honour it.

## Verified against the original

Run side by side on the same host, same moment:

- **53 of 53** services discovered — identical set, no drift either direction.
- **48 of 53** rows byte-identical across every CSV column.
- **Every logging-classification field identical**: `log_status`,
  `log_destination`, `standard_output`, `standard_error`, `effective_output`,
  `effective_error`, `runs_as`, `unit_path`, `description`, `syslog_identifier`.
- **Host posture identical** on all eight fields, and the same four findings.

## Known differences

**1. Runtime state of `oneshot` services (the 5 differing rows).**
`service_facts` collapses systemd's `ActiveState`/`SubState` pair into one
coarse `state`, and reports `stopped` for a `Type=oneshot` unit with
`RemainAfterExit=yes` that systemd considers `active (exited)`. Those units
show as `inactive / dead` here and `active / exited` in the original. The
mapping in `service_audit_native_state_map` recovers the vocabulary for every
other state, but this case is not recoverable from the available facts.

No finding depends on these columns — the logging assessment is unaffected.

**2. `audit_rule_count` means something slightly different.**
The original counts rules *loaded in the kernel* (`auditctl -l`). This role
counts rules *configured on disk* in `/etc/audit/rules.d/*.rules`. They
normally agree, but they will diverge if rules were loaded manually without
being persisted, or persisted without being loaded. Arguably the on-disk count
is the better audit evidence; it is simply not the same measurement.

**3. Declared configuration, not systemd's resolved view.**
`systemctl show` reports what systemd actually computed. This role reads what
the unit files declare and applies the manager default where nothing is
declared. That reproduces the resolved value in every case observed, but it
cannot account for behaviour systemd synthesises at runtime that is not
expressible in the unit files it read.

**4. More round trips.**
The original needs two commands per host regardless of service count. This role
issues one `slurp` per enabled unit file — about 53 on a typical RHEL host.
Set `service_audit_native_read_descriptions: false` to slurp only the handful
of units that actually declare an output directive; service descriptions then
fall back to the unit name.

## On idempotency

Both roles are read-only and report no changes — every `command` task in the
original carries `changed_when: false`. The `changed` entries in a run come
from the `audit_report` role writing a new timestamped evidence file, which is
deliberate: each run is meant to produce a distinct dated artefact.

The case for this role is not idempotency but the things that follow from using
modules: structured return values instead of parsed text, no shell quoting, and
no dependency on the output format of a CLI that is free to change between
systemd releases.
