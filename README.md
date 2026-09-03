# Handshake

Patterns for onboarding an AI coding agent onto a new machine: a first laptop, a replacement machine, a teammate's machine, a fresh VM or cloud box. This repo carries concepts only: the ideas are reusable, the implementation lives wherever your actual setup lives.

The core claim: an agent's identity is reconstructible from three git repos plus a secrets rail, so onboarding is a clone, not a rebuild. Migrating between two machines is one instance of onboarding, the one where a maintaining agent on the existing machine assists, but the same patterns cover every fresh environment.

## The three-repo identity model

An AI coding agent's persistent identity splits cleanly into three git repos:

1. **Skills**: how it does things. Slash commands, workflows, codified procedures.
2. **Memory**: what it knows. Accumulated context, preferences, project state, lessons.
3. **Setup**: how it is configured. Settings, hooks, agent definitions, and a bootstrap script that wires everything together on a fresh machine.

These three repos are the onboarding payload. If all three live on a git host, any new machine can reconstruct "the same agent" with a clone. The machine becomes disposable; the identity is portable. Everything that matters either lives in one of the three repos or is listed by them (see the secrets rail below).

The bootstrap script belongs in the setup repo because setup is the entry point: clone setup, run bootstrap, and bootstrap clones the other two and links everything into place.

## Secrets ride a separate rail

Credentials never enter any of the three repos. Not encrypted, not "temporarily", not in history.

Secrets move point to point: an encrypted tailnet transfer, AirDrop, a password manager, or any channel that never touches the git host. What the repos carry is the **map**, a documented list of which secrets exist and where each one lives on disk, so the onboarding agent knows exactly what to request and where each value goes. Example (invented paths for illustration):

```
~/.config/tool-a/credentials      # API key for tool A
~/projects/.service-b/.env        # service B token
~/.ssh/id_ed25519                 # SSH key, regenerate rather than move if possible
```

The map is safe to publish inside a private setup repo; the values never are. An onboarding run that reaches a secret-dependent step checks the path, and if the file is missing, reports it as skipped rather than failing silently or inventing a placeholder.

## SSO-first auth ordering

A fresh machine needs a long series of browser sign-ins: git host, cloud consoles, SaaS dashboards. Done ad hoc, each one is a password hunt. Done in dependency order, almost all of them collapse into a single approve click.

Bundle every day-one browser authentication into one script, ordered by dependency, with the identity provider sign-in first. That first sign-in is the only real password and 2FA moment. Every service behind the identity provider then authenticates with one click on "continue as you". The script opens each URL in sequence and waits for the human to confirm before moving to the next.

Order rule: identity provider, then git host (everything clones through it), then anything that gates other tools, then the long tail.

## The machine also has to feel right

A machine can pass every functional check and still be the wrong machine to work on. Stock
defaults fight habits: key repeat is slow, autocorrect mangles commit messages, the trackpad
needs a full press, selecting text in the terminal silently overwrites the clipboard. None of
these break a command, so none of them show up in a bootstrap's exit codes, and each one gets
rediscovered and hand-fixed on every new machine.

So environment configuration is part of the onboarding payload, not a personal touch applied
afterwards. It arrives in two shapes, and a setup that only handles the first is half a setup:

1. **Settings the OS exposes through an API**, writable by script. These belong in a declarative
   list the setup repo owns: one row per setting, with the desired value and a note on where the
   value came from.
2. **Per-app config files**, which no OS settings API can reach. These belong in the setup repo
   as tracked files that bootstrap copies into place.

Two rules keep the second kind from rotting:

- **The repo copy is the source of truth; the installed copy is disposable.** Fix a setting in
  the repo and let every machine inherit it, rather than fixing it on the machine in front of
  you. A machine-local edit is a fix with a lifespan of one laptop.
- **Verify the key, not just the file.** Most config formats ignore a misspelled key silently,
  so a "fixed" setting can be no setting at all. Prefer a config language with a validator, and
  make bootstrap assert the observable effect the way it does for everything else.

The trigger for adding something here is worth naming, because it usually arrives as an
annoyance rather than a task: **when a stock default fights a habit, that is a setup defect, not
a one-time fix.** The moment of noticing is the moment to push a row or a file to the setup repo.
Fixing it only on the current machine guarantees meeting it again on the next one.

## Issues as the onboarding channel

The onboarding agent on the new machine files defects and questions as GitHub issues on the setup repo. A maintainer, agent or human, on an established machine answers there and fixes at the source, so every future onboarding benefits. Issues give the channel history, threading, and visibility for free.

Protocol:

- The onboarding agent files an issue when it hits a defect. The issue carries the exact failing command, the exact output, a suggested fix, and which machine it ran on.
- The maintainer watches the repo, fixes the defect at the source (the setup repo, so the fix helps every future machine), and answers in issue comments.
- The onboarding agent **polls rather than blocks**: it moves on to independent steps and checks back, rather than idling on one blocked step.
- Workarounds are allowed but must be documented in the issue, so the maintainer knows the new machine's state diverges until the real fix lands.

The issue tracker outlives both sessions, so an onboarding interrupted mid-way resumes from the open issues, not from anyone's memory.

## Honest onboarding report

An onboarding that "completed" but silently left something broken is worse than a loud failure, because nobody looks for a problem a green checkmark says is not there.

Verify by observable effect, not by exit code:

- Memory installed means memory **actually loads in a fresh session**, not that the clone returned 0.
- A hook installed means the hook **actually fires** on its trigger event, not that the file exists.
- A credential placed means an authenticated call **succeeds**, not that the file has the right name.

The onboarding finishes with a three-bucket report:

1. **Verified**: installed and observed working.
2. **Installed but unverified**: in place, but the observable check was not possible yet (say, a hook whose trigger has not occurred). Named explicitly so a human or later session can close the loop.
3. **Skipped**: not attempted, with the reason (missing secret, open issue, deliberate deferral).

Anything not in bucket 1 is an open item, never a footnote.

## Set expectations for the human

Onboarding runs mostly unattended, so the human's only real question is "do I need to be here, and when should I come back?" The onboarding agent keeps a status file whose header answers that at a glance:

- **A fixed-denominator progress line** ("10 of 28 tasks complete"): the task list is written down before the run starts and never shrinks mid-run, so the number means the same thing on every machine.
- **A "human needed" line that batches attention**: browser sign-ins cluster into one authentication pass, announced as a block ("come back in about 5 minutes for about six clicks"), which beats a ping per sign-in.
- **A "now" line** saying what the agent is doing at this moment.

Pushing the status file to the setup repo doubles as the check-in: anyone watching the repo sees progress without asking.

## Instrument every run

Onboarding improves run over run only if runs are comparable. The status file records machine details once (chip, memory, OS version), elapsed time per task inline, and token spend at each phase boundary. When a step runs slower or costs more than it did on the previous machine, that difference is a defect report waiting to be filed, not a shrug.

## The one-machine rule for scheduled workers

Cron and launchd jobs that append to shared destinations (a spreadsheet, a database, an API with side effects) must run on exactly one machine. Two machines running the same appender produce duplicate rows, double sends, and rate-limit collisions that look like flaky infrastructure.

Onboarding therefore never copies a scheduled job onto the new machine. When ownership should move, the transfer is **unload-then-load, never copy**: disable the job on the owning machine, confirm it is disabled, then enable it on the new machine. The window where neither machine runs it is safe (one missed tick); the window where both run it is not.

The setup repo lists which workers exist and which machine currently owns each one, so ownership is a recorded fact rather than tribal knowledge.

## Concurrent editors

During an onboarding, more than one agent may work the setup repo at once: the onboarding agent, a maintainer, other sessions on established machines. Assume it; do not treat a conflict as a surprise.

- **Prefer new files over edits** where either would do. Two agents adding two new files never conflict; two agents editing the same file often do.
- **Pull before push**, every time. Another machine probably pushed since you last looked.
- **Name your machine in commits** (in the message or a trailer), so the history reads as a conversation between machines rather than an anonymous stream.

These are the same habits any distributed team uses; the only shift is applying them to agents by default rather than hoping a single-writer assumption holds.

## Supporting docs

- [docs/bootstrap-report.md](docs/bootstrap-report.md): example of the three-bucket report format.
- [docs/defect-issue.md](docs/defect-issue.md): example of a defect issue filed by the onboarding agent.

## License

MIT, see [LICENSE](LICENSE).
