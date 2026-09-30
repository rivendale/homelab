# Alerting

Prometheus alert rules, a push notification server (ntfy), and watchdog timers on a small home
lab.

## Already covered elsewhere

These are in the [opensource lessons](https://github.com/rivendale/opensource/blob/main/lessons/README.md)
and are not repeated here:

- [Identical alerts destroy the signal](https://github.com/rivendale/opensource/blob/main/lessons/README.md#monitoring-and-alerts):
  key alerts on the failure shape and carry a suppressed count.
- [An alert whose wait outlasts the outage never fires](https://github.com/rivendale/opensource/blob/main/lessons/README.md#monitoring-and-alerts):
  set each `for:` below the shortest outage that must page.
- [Assert what the response contains](https://github.com/rivendale/opensource/blob/main/lessons/README.md#deploying-small-games):
  `curl -f` exits 0 on a redirect.
- [Compare against a shared ref, not the checkout](https://github.com/rivendale/opensource/blob/main/lessons/README.md#testing-prove-each-check-can-fail):
  a drift monitor that reads the working tree reports on whichever branch is checked out.
  One addition from 2026-08: the same mistake happens by hand. `git log` on someone else's
  checkout shows that branch, not what is merged; `git merge-base --is-ancestor <sha> origin/main`
  settles it.

## ntfy turns a long message into an attachment

**ntfy stores any message over 4096 bytes as an attachment, replaces the text with "You received a
file: attachment.json", and still returns 200.**

- **Failure (2026-08):** messages between machines went through ntfy. Long ones were stored as
  attachments; the sender's `curl -fsS ... && echo sent` printed success and every reader that
  parsed the body as JSON skipped them. It went unnoticed all day.
- **Measured on:** ntfy's default `message-size-limit` of 4096 bytes, 2026-08. The limit applies
  to the encoded payload, so a body with many newlines crosses it before its raw length does.
- **Check:** after a publish, check the response JSON for an `attachment` field instead of trusting
  the status code. To find past drops, poll `?poll=1&since=all` and count messages whose body is
  not what your readers expect. A reader's normal poll filters them out.
- **Recovery:** the content is at `https://<ntfy-host>/file/<message-id>.json` until attachments
  expire (three hours by default). Fix the sender by splitting on encoded size into
  self-contained numbered parts.

## Filter on an optional label with negation

**A selector on a label that some series may lack must use `!=`, not `=`; equality matches nothing
on a series without the label, so the rule goes blind while reading healthy.**

- **Failure avoided (2026-09):** a `scope` label was added to a health metric so one host's check
  of another's endpoint stopped paging as its own health. The rule was written as
  `health_check_ok{scope!="external"} == 0`, not `scope="self"`. Proved both ways: without the
  matcher the external check paged again, and with equality an unlabeled series went silent. A
  rollback, an older build or one host on the previous collector all produce series with no label.
- **Measured on:** Prometheus alert rules, 2026-09.
- **Check:** for each matcher on a recently added label, ask what the rule does for a series that
  lacks it. Fail toward the previous, broader behavior. When publishing a metric, add labels and
  never remove or rename them.

## Test a suppression on release

**An alert that defers to its root cause must be tested when the cause clears and when the cause
metric is absent, not only while it is firing.**

- **Why (2026-09):** a suppression that silences a symptom while its cause is alerting can also
  silence it forever if the cause series disappears. It must fire again when the cause is fixed and
  fire when the cause metric is missing entirely.
- **Check:** in a rule test or on a test instance, remove the cause series and confirm the symptom
  alert fires.

## A watchdog on the same host cannot report its death

**A monitor running on the machine it watches can report degradation of that machine, never its
death.**

- **Failure (2026-08):** a WSL distro shut down when its last terminal closed. The watchdog timer,
  the health service and the unit-failure notifier all lived inside it and died with it: 27 minutes
  dark, zero notifications. The only thing that could see it was Prometheus on another machine, and
  that rule's `for:` was longer than the outage.
- **Check:** for every monitor, ask what failure takes the monitor with it. Host death must be
  detected from another host, with an absence rule (`up == 0` or `absent(...)`) whose `for:` is
  shorter than the outages you care about.

## One sample of a volatile signal is not evidence

**A metric that swings widely on a healthy system cannot trigger a restart from a single reading.**

- **Failure (2026-08):** a watchdog restarted a healthy service four times in 28 minutes. A
  service's established socket count, measured between 2 and 67 within three minutes when healthy,
  read zero for a moment when a network adapter flapped. The check that was meant to corroborate
  had become the sole trigger. Separately, its primary signal read the wrong session's log file,
  because two sessions shared one directory and it picked the newest file.
- **Check:** require consecutive bad readings before acting, publish the streak as its own metric
  so near misses stay visible, and select the file or process by an identifying record, not by
  "newest".

## A textfile metric can be stale

**A Prometheus textfile-collector file keeps being scraped after the job that writes it stops, so
its values can be days old and still read green.**

- **Why (2026-08):** health results written by a cron job into the node exporter's textfile
  directory are only as current as the last run.
- **Check:** have the writer include a `*_last_run_timestamp_seconds` metric, and alert on
  `time() - <metric> > <interval>` as well as on the results.
