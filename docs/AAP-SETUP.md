# Setting this up in AAP

Written against AAP 2.7 (Controller 4.8.11). Work through it in order — the
baseline has to exist before drift detection means anything.

## 1. Project

| Field | Value |
|---|---|
| Name | `SOX Service Audit` |
| Organization | the org your Linux inventory lives in |
| Source Control Type | Git |
| Source Control URL | `https://github.com/joebrown-RH/service_enabled.git` |
| Branch | `main` |
| Options | Update Revision on Launch |

No SCM credential needed — the repo is public. There is deliberately no
`collections/requirements.yml`, so the sync is a plain git clone with no Galaxy
step; if a sync takes minutes, something else is wrong.

## 2. Where reports land — set this once

Set these as **variables on the AAP inventory** (Inventories → your inventory →
Variables), not per job template, so every template inherits them:

```yaml
sox_report_host: reports              # a durable host, not the EE
sox_report_dir: /var/www/html/reports # the product-demos report server docroot
sox_report_become: true               # the web root is root-owned
```

Three things that cost time if you don't know them:

- **Left unset, reports render inside the execution environment** and are
  destroyed with the job. You get a green job and no artefact.
- **If the same variables also appear in a job template's Variables field, the
  template wins** — that field is extra vars. Set them in one place.
- **The job template's Privilege Escalation checkbox does not cover the report
  write.** The report play runs `become: false` and takes escalation solely
  from `sox_report_become`.

The project's `inventory/group_vars/all.yml` is **not** read by AAP — AAP uses
the Controller inventory. That file only affects local `ansible-playbook` runs.

Reports land at `http://<report-host>/reports/sox_service_audit_latest.html`.
They also join the product-demos reports landing page, which is a static index
built by a `find` for `*.html` — so a new report appears there the next time
that role runs, not instantly.

## 3. Job templates

All four share: Project `SOX Service Audit`, your Linux inventory, your machine
credential, **Privilege Escalation on**.

| Name | Playbook | Notes |
|---|---|---|
| `SOX - Collect Service & Logging Evidence` | `playbooks/sox_service_audit.yml` | `sox_hosts: all`, or your group name |
| `SOX - Capture Config Baseline` | `playbooks/drift_baseline.yml` | run this before the sensor |
| `SOX - Drift Sensor` | `playbooks/drift_sensor.yml` | needs the Event Stream vars from step 5 |
| `SOX - Drift Response` | `playbooks/drift_response.yml` | **Prompt on launch** for Variables *and* Limit |

Drift Response is prompt-on-launch because EDA passes the drifted host and the
change details in at launch time.

Privilege escalation genuinely matters for the audit: without root, auditd rule
counts come back `0` and root-owned configs record as `unreadable`, so the
report looks artificially clean.

## 4. Baseline first

Launch **SOX - Capture Config Baseline** once against every in-scope host.
Nothing downstream works until each host has `/var/lib/sox_audit/baseline.json`.
Drift is measured against an approved state, not against yesterday.

Re-run it after any authorised change, or that change reports as drift forever.

## 5. Event-Driven Ansible

**Decision Environment** — the supplied `Default Decision Environment` is fine.
The rulebook needs nothing beyond `ansible.eda`, which every DE ships.

**Project** → Automation Decisions → Projects → Create, same Git URL and branch
as step 1. This is a separate object from the Controller project.

**Credential** → Automation Decisions → Credentials → type **Red Hat Ansible
Automation Platform**, pointing at the controller with a token. Without it the
rulebook cannot launch job templates.

**Event Stream** → Automation Decisions → Event Streams → Create, type
**Webhook**, with a Token or Basic credential. It returns a URL shaped like:

```
https://<gateway>/eda-event-streams/api/eda/v1/external_event_stream/<uuid>/post/
```

> On AAP 2.5 and later an activation's webhook source is **not reachable
> directly** — the activation runs in a container with no external route.
> Inbound events have to arrive through an Event Stream. This is the step most
> likely to catch you out if you have done EDA demos on 2.4.

**Rulebook Activation** → Create:

- Project: the EDA project from above
- Rulebook: `config_drift_webhook.yml`
- Decision Environment, plus the AAP credential
- Attach the Event Stream and map it to the rulebook's webhook source

## 6. Point the sensor at the Event Stream

On **SOX - Drift Sensor**, add:

```yaml
eda_webhook_url: "<the Event Stream URL>"
eda_webhook_token: "<token>"        # or eda_webhook_user / eda_webhook_password
```

While wiring this up, add `eda_webhook_no_log: false` so the request and
response are visible in the job output. Take it back out afterwards — it prints
the Authorization header.

## 7. Prove it end to end

```bash
sudo systemctl stop firewalld        # on one target host
```

Launch **SOX - Drift Sensor**. It should report drift, post to the Event
Stream, and the activation should launch **SOX - Drift Response**, which shows
the diff against the approved copy, the current state of the service, the
auditd attribution for who did it, and appends a row to `drift_register.csv`
alongside the reports.

Then re-enable the schedule in `config_as_code/controller_templates.yml` (it
ships disabled) to run the sensor every 10 minutes.

## If the activation does not fire

The sensor posting successfully while nothing launches means the rulebook's
conditions do not match the event shape. The rulebook reads
`event.payload.source` and `event.payload.drift.*`, which is correct for a raw
`ansible.eda.webhook` source; an Event Stream may nest the body differently.

Open the activation's **History → output**, look at how the event is actually
structured, and adjust the two `condition:` lines in
`rulebooks/config_drift_webhook.yml` to match. This is unverified against a
live Event Stream — it is the one step in this document that has not been
tested end to end.

## Known environment issue

The AAP MCP server deployed on this cluster (`aapmcp` in the `aap` namespace)
answers gateway endpoints but returns HTML `404 Not Found` for every Controller
and EDA endpoint. Its bundled `data/controller-schema.json` declares paths as
`/api/v2/...`, which AAP 2.7's gateway serves at `/api/controller/v2/...`.
Nothing in `aap-mcp.yaml` overrides the prefix, so this is a packaging bug in
the `mcp-server-rhel9` image, not a configuration problem. It does not affect
any of the automation in this repo — only tooling that drives AAP through MCP.
