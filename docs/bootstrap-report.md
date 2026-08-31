# Bootstrap report format

The bootstrap ends with a three-bucket report: verified, installed but unverified, and skipped. Every line names the observable check that was (or was not) run. The report below is an invented example for illustration.

```
Bootstrap report, machine: laptop-new, 2026-01-15

Verified
- skills repo cloned and linked: /skills-list shows 42 skills in a fresh session
- memory repo cloned and linked: fresh session quotes a known memory entry
- git host auth: authenticated API call returned the expected username
- settings applied: fresh session reflects the configured model and permissions

Installed but unverified
- session-start hook: file in place and registered, fires on next fresh
  session start, not yet observed

Skipped
- cloud CLI credentials: secret file not yet transferred (see secrets map,
  item 3); blocked, issue #12 filed
- scheduled worker "daily-sync": deliberately deferred, old machine still
  owns it (one-machine rule); unload-then-load scheduled for cutover day
```

Rules the example follows:

- Each verified line states the effect observed, not the command that ran.
- Each unverified line states what would verify it and why that has not happened yet.
- Each skipped line states the reason and points at the tracking artifact (issue, secrets map entry).
