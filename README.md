# SOX Evidence: Linux Enabled Services & Logging Posture

Ansible automation that answers two questions a SOX auditor will ask about every
Linux host in scope, and keeps answering them after the auditor leaves:

1. **What starts automatically on this host, and who runs it?**
   Every `enabled` systemd service, its state, and the account it runs as.
2. **If something happened, would we have a log of it?**
   For each service, whether its output actually reaches a durable log sink --
   and for each host, whether the journal is persistent, forwarded off-box,
   covered by auditd, and retained for a defined period.

The output is a self-contained HTML report for the audit pack and a flat CSV for
the auditor's working papers. A second, event-driven half of the demo shows the
platform noticing when any of that drifts.

---

## What the report looks like

Six headline tiles, a logging-coverage bar per host, the control findings in
plain language, the host logging posture, and a filterable table of every
enabled service with an assessment of each.

Every status is shown as **an icon plus a word**, never colour alone -- so the
report survives being printed in black and white, photocopied into an audit
binder, and read by someone with colour-vision deficiency.

---

## Repository layout

```
playbooks/
  sox_service_audit.yml    Collect the evidence and render the report
  drift_baseline.yml       Record the approved "known good" state
  drift_sensor.yml         Compare against baseline, raise an EDA event
  drift_response.yml       What EDA runs when a host drifts
roles/
  service_audit/           Collection and assessment logic
  audit_report/            HTML + CSV rendering
rulebooks/
  config_drift_webhook.yml Fleet-wide drift via the sensor (use this in AAP)
  config_drift_local.yml   Live file/service watching (laptop or single VM)
config_as_code/            Job templates, schedule, project, EDA activation
inventory/                 Demo inventory; replace with your AAP inventory
```

---

## Quick start

```bash
ansible-galaxy collection install -r collections/requirements.yml

# Audit the control node itself -- the fastest way to see real output
ansible-playbook playbooks/sox_service_audit.yml -e sox_hosts=local_demo -K

# Then open:
open reports/sox_service_audit_latest.html
```

Against real targets, put the hosts in the `sox_targets` group and run:

```bash
ansible-playbook playbooks/sox_service_audit.yml
```

> **Run it with `become`.** Without root the audit still completes, but auditd
> rule counts come back as zero and root-only config files are recorded as
> `unreadable` rather than fingerprinted. The playbooks set `become: true`; you
> just need the privilege escalation to actually work.

---

## How a service is assessed

For every enabled unit the role reads `StandardOutput`, `StandardError`,
`SyslogIdentifier` and `SyslogFacility` from systemd, resolving `inherit`
against the manager defaults, then classifies it:

| Verdict | Meaning |
|---|---|
| **Captured** | stdout reaches `journal`, `syslog` or `kmsg`, and stderr is not discarded. |
| **Review** | Output goes to a file or console, or stderr is discarded. It exists, but it is not in the central pipeline and needs a human decision. |
| **Not captured** | `StandardOutput=null`. The service produces no log evidence at all. |

At host level the role records whether the journal is persistent, its configured
retention and size cap, whether rsyslog forwards off-box (with the matching
config lines included as evidence, not just a yes/no), whether auditd is enabled
and how many rules are loaded, and whether logrotate is configured. Anything
missing becomes a finding, phrased the way a reviewer would write it up.

The collection is deliberately cheap: one `systemctl show` call per host covers
every enabled unit, rather than one call per service.

---

## The config drift demo

Drift is measured against an explicitly approved baseline rather than against
"whatever it looked like last time":

```bash
ansible-playbook playbooks/drift_baseline.yml     # approve the current state
ansible-playbook playbooks/drift_sensor.yml -e eda_webhook_url=http://eda-host:5000/endpoint
```

The sensor fingerprints the watched config files and the enabled-service set,
compares them to the baseline, and posts one event per drifted host. The
rulebook launches **SOX - Drift Response**, which gathers the evidence --
a real diff against the approved copy of the file, the current state of any
service that changed, and the auditd attribution for who touched it -- and
appends the event to `reports/drift_register.csv`.

The response playbook is **non-destructive by design**. It documents the change;
it does not revert it. Auto-remediation of a SOX-relevant control is a decision
a customer should make deliberately, not a default.

### Two rulebooks, and why

| | `config_drift_webhook.yml` | `config_drift_local.yml` |
|---|---|---|
| Source | webhook, fed by `drift_sensor.yml` | `file_watch` + `journald` |
| Scope | the whole fleet | the one machine the rulebook runs on |
| Where it runs | an AAP EDA activation | `ansible-rulebook` on a RHEL box |

**This distinction matters for the demo.** An EDA activation in AAP runs inside a
decision environment *container*. A `file_watch` source there watches the
container's filesystem, not your managed hosts -- so it will never see someone
edit `/etc/ssh/sshd_config` on a server. That is why the fleet-wide path senses
on the hosts and pushes events in.

`config_drift_local.yml` is the one to use for the visceral "watch me break it"
moment on a single VM:

```bash
ansible-rulebook --rulebook rulebooks/config_drift_local.yml \
                 --inventory inventory/hosts.yml -v

# in another terminal on that host:
sudo sed -i 's/^#MaxAuthTries.*/MaxAuthTries 10/' /etc/ssh/sshd_config
sudo systemctl stop firewalld
```

Both rules are throttled to once per 30 seconds per file/unit, because editors
write a file several times per save and an unthrottled demo turns into a storm.

---

## Running it in AAP

`config_as_code/` defines the project, four job templates, a (disabled) schedule
for the sensor, and the EDA activation, in the `infra.aap_configuration` format.
Edit `config_as_code/vars.yml` to match your instance first -- organization,
inventory, execution environment, credential and decision environment names.

Suggested demo flow in the UI: **Capture Config Baseline** -> **Collect Service
& Logging Evidence** (show the report) -> break something -> **Drift Sensor** ->
watch the activation fire **Drift Response**.

### Getting the report out of AAP

The report is written inside the execution environment, which is discarded when
the job ends. For anything beyond a live demo, send it somewhere durable --
point `sox_report_dir` at a mounted volume on a container group, add a task to
push it to object storage or a web server, or email it from the final play.
Plan for this before showing it to a customer who asks "so where does the PDF
land?"

---

## Scope and honest limitations

- **systemd only.** The audit asserts `service_mgr == 'systemd'` and stops
  otherwise. RHEL 8/9/10 and derivatives.
- **This is evidence collection, not a control framework.** It gathers what an
  ITGC logging/service control needs; it does not assert that your controls are
  adequate, and it is not a CIS or STIG benchmark.
- **"Captured" is about plumbing, not content.** The report tells you a service's
  output reaches the journal. It does not judge whether the service logs the
  right events at the right verbosity.
- **Baseline copies hold config content.** `drift_baseline.yml` keeps a root-only
  (`0600`) copy of each watched file on the host it came from, so the drift
  response can show a real diff. Set `sox_baseline_keep_copies=false` to disable.

---

## Verification

`ansible-lint` passes at the `production` profile. The collection logic, report
rendering, baseline capture, drift detection and the webhook payload have all
been run end to end against a live systemd host.
