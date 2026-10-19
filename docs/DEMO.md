# Run of show

A 20-minute customer demo. Times are a guide, not a script.

## Before the customer joins

```bash
ansible-playbook playbooks/drift_baseline.yml          # approve current state
ansible-playbook playbooks/sox_service_audit.yml       # warm run, check it is clean
```

Have ready: the AAP UI on the templates list, a terminal on one target host, and
`reports/sox_service_audit_latest.html` open in a browser tab you can refresh.

If you are demoing the live file-watch, start the rulebook now in its own
terminal so it is already listening:

```bash
ansible-rulebook --rulebook rulebooks/config_drift_local.yml \
                 --inventory inventory/hosts.yml -v
```

---

## 1. Frame the problem (2 min)

Two questions the auditor asks every year, that someone currently answers with a
spreadsheet and a week of screenshots:

- What runs automatically on these servers, and as whom?
- If something happened on them, would we have a log of it?

Point out that the second question is the one that usually fails. Everyone can
list services. Far fewer can prove the output of those services is retained,
forwarded off-box, and covered by an audit trail.

## 2. Collect the evidence (5 min)

Launch **SOX - Collect Service & Logging Evidence**.

While it runs, say what it is doing: one `systemctl show` call per host covering
every enabled unit, rather than hundreds of round trips. It scales to a fleet.

Open the report and walk the top down:

- **The tiles.** Hosts in scope, enabled services, and the split between
  captured / review / not captured.
- **The coverage bar.** Mostly green is the expected, reassuring shape. Then
  point at the red sliver.
- **The findings.** Read one aloud -- they are written the way a reviewer would
  write them, not as raw data.
- **Filter the table to "Not captured."** This is the moment. A service with
  `StandardOutput=null` is running, enabled, in scope, and producing no evidence
  whatsoever. Nobody finds that in a spreadsheet.

Then show the CSV and note that it pivots straight into the auditor's own
working papers -- you are not asking them to adopt a new tool.

## 3. Show that it stays true (8 min)

The audit is a point in time. The control question is what happens on day 2.

Break something, visibly:

```bash
sudo sed -i 's/^#MaxAuthTries.*/MaxAuthTries 10/' /etc/ssh/sshd_config
sudo systemctl stop firewalld
```

- **Local rulebook path:** the terminal reacts within seconds.
- **AAP path:** launch **SOX - Drift Sensor** (or let the schedule fire). It
  posts to the EDA activation, which launches **SOX - Drift Response**.

In the drift response job output, show the three things that matter:

1. The **diff** against the approved copy -- not "a checksum changed", the
   actual line.
2. The **current state** of the service that moved.
3. The **auditd attribution** -- which account did it.

Then show `reports/drift_register.csv` accumulating a dated record.

Make the point explicitly: **the response documents, it does not revert.**
Auto-remediating a SOX-relevant control should be the customer's deliberate
choice, and the moment you say that out loud, the compliance people in the room
relax.

## 4. Close (3 min)

- Baseline is explicit and version-controlled. Drift means "differs from what we
  approved", not "differs from yesterday".
- The whole thing is four job templates and one rulebook, defined as code in
  `config_as_code/`.
- Re-running the audit after the change regenerates the evidence pack, so the
  audit trail and the current state never diverge.

---

## Questions you will get

**"Does this change anything on my servers?"**
The audit is read-only. `drift_baseline.yml` writes to `/var/lib/sox_audit/` on
each host, and nothing else writes anywhere.

**"Where does the report actually land?"**
By default, inside the execution environment, which is destroyed when the job
ends -- fine for this demo, useless as an artefact. Set `sox_report_host` to a
durable host in their inventory and it writes there instead, with a stable
`*_latest.html` to point a dashboard at. Say this before they ask it; it lands
much better as a design decision than as a gap you got caught on.

**"Can it fix the problems it finds?"**
Yes, but deliberately not by default. See the close above.

**"Why not just use file_watch against all my servers?"**
Because an EDA activation runs in a container and would be watching that
container. That is exactly why the fleet path senses on the hosts and pushes
events in. This question comes up often enough to be worth pre-empting.

**"Is this a CIS/STIG benchmark?"**
No. It is evidence collection for an ITGC logging and service control. It
complements a benchmark scan rather than replacing one.
