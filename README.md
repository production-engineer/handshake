# Handshake

Patterns for migrating an AI coding agent between machines, with agent sessions on both machines cooperating. This repo carries concepts only: the ideas are reusable, the implementation lives wherever your actual setup lives.

The core claim: an agent's identity is reconstructible from three git repos plus a secrets rail, and the migration itself is a job two agents can run together, one bootstrapping the new machine, one maintaining the repos from the old machine.

## The three-repo identity model

An AI coding agent's persistent identity splits cleanly into three git repos:

1. **Skills**: how it does things. Slash commands, workflows, codified procedures.
2. **Memory**: what it knows. Accumulated context, preferences, project state, lessons.
3. **Setup**: how it is configured. Settings, hooks, agent definitions, and a bootstrap script that wires everything together on a fresh machine.

If all three live on a git host, any new machine can reconstruct "the same agent" with a clone. The machine becomes disposable; the identity is portable. Everything that matters either lives in one of the three repos or is listed by them (see the secrets rail below).

The bootstrap script belongs in the setup repo because setup is the entry point: clone setup, run bootstrap, and bootstrap clones the other two and links everything into place.

## Secrets ride a separate rail

Credentials never enter any of the three repos. Not encrypted, not "temporarily", not in history.

Secrets move point to point between machines: an encrypted tailnet transfer, AirDrop, or any channel that never touches the git host. What the repos carry is the **map**, a documented list of which secrets exist and where each one lives on disk, so the bootstrap agent knows what to ask for and where to put it. Example (invented paths for illustration):

```
~/.config/tool-a/credentials      # API key for tool A
~/projects/.service-b/.env        # service B token
~/.ssh/id_ed25519                 # SSH key, regenerate rather than move if possible
```

The map is safe to publish inside a private setup repo; the values never are. A bootstrap that reaches a secret-dependent step checks the path, and if the file is missing, reports it as skipped rather than failing silently or inventing a placeholder.

## SSO-first auth ordering

A fresh machine needs a long series of browser sign-ins: git host, cloud consoles, SaaS dashboards. Done ad hoc, each one is a password hunt. Done in dependency order, almost all of them collapse into a single approve click.

Bundle every browser authentication into one script, ordered by dependency, with the identity provider sign-in first. That first sign-in is the only real password and 2FA moment. Every service behind the identity provider then authenticates with one click on "continue as you". The script opens each URL in sequence and waits for the human to confirm before moving to the next.

Order rule: identity provider, then git host (everything clones through it), then anything that gates other tools, then the long tail.

## Issues as an agent-to-agent channel

Two agents work the migration at once: a bootstrapping agent on the new machine and a maintaining agent on the old machine. They coordinate through GitHub issues on the setup repo, which gives the channel history, threading, and visibility for free.

Protocol:

- The bootstrapping agent files an issue when it hits a defect. The issue carries the exact failing command, the exact output, a suggested fix, and which machine it ran on.
- The maintaining agent watches the repo, fixes the defect at the source (the setup repo, so the fix helps every future machine), and answers in issue comments.
- The bootstrapping agent **polls rather than blocks**: it moves on to independent steps and checks back, rather than idling on one blocked step.
- Workarounds are allowed but must be documented in the issue, so the maintaining agent knows the new machine's state diverges until the real fix lands.

The issue tracker outlives both sessions, so a migration interrupted mid-way resumes from the open issues, not from anyone's memory.

## Honest bootstrap reporting

A bootstrap that "completed" but silently left something broken is worse than a loud failure, because nobody looks for a problem a green checkmark says is not there.

Verify by observable effect, not by exit code:

- Memory installed means memory **actually loads in a fresh session**, not that the clone returned 0.
- A hook installed means the hook **actually fires** on its trigger event, not that the file exists.
- A credential placed means an authenticated call **succeeds**, not that the file has the right name.

The bootstrap finishes with a three-bucket report:

1. **Verified**: installed and observed working.
2. **Installed but unverified**: in place, but the observable check was not possible yet (say, a hook whose trigger has not occurred). Named explicitly so a human or later session can close the loop.
3. **Skipped**: not attempted, with the reason (missing secret, open issue, deliberate deferral).

Anything not in bucket 1 is an open item, never a footnote.

## The one-machine rule for scheduled workers

Cron and launchd jobs that append to shared destinations (a spreadsheet, a database, an API with side effects) must run on exactly one machine. Two machines running the same appender produce duplicate rows, double sends, and rate-limit collisions that look like flaky infrastructure.

Migration of a scheduled worker is therefore **unload-then-load, never copy**: disable the job on the old machine, confirm it is disabled, then enable it on the new machine. The window where neither machine runs it is safe (one missed tick); the window where both run it is not.

The setup repo lists which workers exist and which machine currently owns each one, so ownership is a recorded fact rather than tribal knowledge.

## Concurrent editors

During a migration, more than one agent works the setup repo at once. Assume it; do not treat a conflict as a surprise.

- **Prefer new files over edits** where either would do. Two agents adding two new files never conflict; two agents editing the same file often do.
- **Pull before push**, every time. The other machine probably pushed since you last looked.
- **Name your machine in commits** (in the message or a trailer), so the history reads as a conversation between machines rather than an anonymous stream.

These are the same habits any distributed team uses; the only shift is applying them to agents by default rather than hoping a single-writer assumption holds.

## Supporting docs

- [docs/bootstrap-report.md](docs/bootstrap-report.md): example of the three-bucket report format.
- [docs/defect-issue.md](docs/defect-issue.md): example of a defect issue filed by the bootstrapping agent.

## License

MIT, see [LICENSE](LICENSE).
