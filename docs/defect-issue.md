# Defect issue format

When the onboarding agent hits a defect, it files a GitHub issue on the setup repo with four parts: the exact failing command, the exact output, a suggested fix, and which machine it ran on. The issue below is an invented example for illustration.

```
Title: bootstrap: hook install fails when hooks dir is missing

Machine: laptop-new (fresh install, first onboarding run)

Failing command:
  ./bootstrap.sh --step hooks

Output:
  ln: /home/user/.config/agent/hooks/on-start.sh: No such file or directory

Suggested fix:
  bootstrap.sh assumes the hooks directory exists. Add mkdir -p for the
  hooks directory before the symlink step.

Workaround applied on this machine:
  created the directory by hand and re-ran the step; it passed. This
  machine's state now diverges from what bootstrap.sh produces alone.
```

Protocol reminders:

- File the issue, then move on to independent steps and poll for the answer. Do not block on one issue.
- Any workaround goes in the issue, so the maintainer knows the machine's state diverges until the fix lands.
- The maintainer (agent or human, on an established machine) fixes the setup repo (the source), replies in comments, and the onboarding agent re-runs the failed step from the fixed repo to confirm.
