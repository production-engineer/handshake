# Watching how much memory your agent sessions use

Each Claude Code session is small, a few hundred MB. The number of sessions is what hurts. People leave sessions open for days in terminal tabs, and twenty of them can quietly take 6 GB or more. macOS doesn't cap how much memory one app can use. When swap runs out, the kernel pauses the terminal app and can then kill it, closing every session inside it at once. Nothing warns you first. The Mac just gets slower until it stops.

So the setup repo installs a small watcher on every machine. It counts the memory all agent processes use together and posts a notification when the total passes a limit.

## What it does and doesn't do

- **It only notifies.** Which session to close is a human call, since some old session may be holding work that matters. Killing processes automatically is easy to add, but it's the wrong default.
- **It adds up all the sessions, not each one separately.** Any one session is usually fine. What crosses the limit is the total.
- **It rate-limits itself.** It checks every 60 seconds and posts at most one notification every 15 minutes, so a busy afternoon doesn't turn into a stream of popups.
- **It runs on every machine.** This is the exception to the one-machine rule for scheduled workers: it reads only the processes on its own Mac and doesn't write anywhere shared, so it belongs on every Mac.

## Picking the limit

Start at about a third of the machine's RAM: 6 GB on a 16 GB Mac, 12 GB on a 36 GB one. Set it with `CLAUDE_MEM_LIMIT_GB`. If you get notifications while the Mac still feels fast, raise it. If the Mac slows down before you get one, lower it.

When the Mac is already slow, check `sysctl vm.swapusage` before you look at CPU. `memory_pressure` can report plenty of free memory while swap is almost full.

## The script

Save it anywhere, make it executable, check it once, then install it:

```
chmod +x claude-mem-watch
./claude-mem-watch --check      # lists sessions by size; changes nothing
CLAUDE_MEM_LIMIT_GB=6 ./claude-mem-watch --install
```

`--check` shows each session with its terminal (`ttys003`) and how long it has been running, so you can find the tab to close. `--install` writes a launchd job that points at the script where it sits now, so put the script somewhere permanent first. To remove it, run `launchctl bootout gui/$UID/local.claude-mem-watch` and delete `~/Library/LaunchAgents/local.claude-mem-watch.plist`.

To check that it works, force the bad state on purpose: `CLAUDE_MEM_LIMIT_GB=0 ./claude-mem-watch` should post a notification right away. If it doesn't show up, allow notifications for Script Editor in System Settings (that's the app `osascript` notifications come from).

```bash
#!/bin/bash
# Notifies when all Claude Code sessions on this Mac together use more memory than it can spare.
#
#   claude-mem-watch            check once, notify if over the limit (what launchd runs)
#   claude-mem-watch --check    print the total and every session by size, never notify
#   claude-mem-watch --install  write and load a launchd job that runs the check every 60s
set -uo pipefail

LABEL="local.claude-mem-watch"
LIMIT_GB="${CLAUDE_MEM_LIMIT_GB:-6}"   # set it to about a third of the Mac's RAM
QUIET_SECS=900                         # at most one notification per 15 minutes
LOGS="$HOME/Library/Logs/claude-mem-watch"
STAMP="$LOGS/last-notified"

if [ "${1:-}" = "--install" ]; then
  self="$(cd "$(dirname "$0")" && pwd)/$(basename "$0")"
  plist="$HOME/Library/LaunchAgents/$LABEL.plist"
  mkdir -p "$HOME/Library/LaunchAgents" "$LOGS"
  cat > "$plist" <<PLIST
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>$LABEL</string>
  <key>ProgramArguments</key><array><string>$self</string></array>
  <key>EnvironmentVariables</key><dict><key>CLAUDE_MEM_LIMIT_GB</key><string>$LIMIT_GB</string></dict>
  <key>RunAtLoad</key><true/>
  <key>StartInterval</key><integer>60</integer>
  <key>StandardOutPath</key><string>$LOGS/launchd.out</string>
  <key>StandardErrorPath</key><string>$LOGS/launchd.err</string>
  <key>ProcessType</key><string>Background</string>
</dict>
</plist>
PLIST
  plutil -lint "$plist" >/dev/null || exit 1
  launchctl bootout "gui/$UID/$LABEL" 2>/dev/null || true
  launchctl bootstrap "gui/$UID" "$plist" && echo "$LABEL loaded (limit ${LIMIT_GB} GB)" || { echo "$LABEL FAILED to load"; exit 1; }
  exit 0
fi

# one row per claude process: rss in KB, pid, tty, elapsed
rows="$(ps -axo rss=,pid=,tty=,etime=,comm= | awk '$5 ~ /(^|\/)claude$/')"
count="$(printf '%s\n' "$rows" | grep -c . || true)"
total_kb="$(printf '%s\n' "$rows" | awk '{s+=$1} END{print s+0}')"
total_gb="$(awk -v k="$total_kb" 'BEGIN{printf "%.1f", k/1048576}')"
top3="$(printf '%s\n' "$rows" | sort -rn | head -3 | awk '{printf "%s %.1fGB up %s; ", $3, $1/1048576, $4}')"

if [ "${1:-}" = "--check" ]; then
  echo "$count sessions, ${total_gb} GB (limit ${LIMIT_GB} GB)"
  printf '%s\n' "$rows" | sort -rn | awk 'NF{printf "  pid %-6s %-8s %5.2f GB  up %s\n", $2, $3, $1/1048576, $4}'
  exit 0
fi

over="$(awk -v t="$total_gb" -v l="$LIMIT_GB" 'BEGIN{print (t>l)?1:0}')"
[ "$over" = 1 ] || exit 0

now="$(date +%s)"
last="$(cat "$STAMP" 2>/dev/null || echo 0)"
[ $((now - last)) -ge $QUIET_SECS ] || exit 0
mkdir -p "$LOGS"; echo "$now" > "$STAMP"

msg="$count sessions use ${total_gb} GB (limit ${LIMIT_GB}). Largest: ${top3%; }"
echo "$(date '+%F %T') $msg"
osascript -e "display notification \"$msg\" with title \"Claude memory\" sound name \"Funk\""
```

## Making each session lighter

The watcher tells you when there are too many sessions. It doesn't make any one of them smaller. One option is to set `BUN_JSC_forceRAMSize` (in bytes, for example `"2147483648"` for 2 GB) in the `env` block of `~/.claude/settings.json`. That tells the runtime to clean up memory as if the machine had only that much RAM. Treat it as an experiment until you've measured a difference. Closing old sessions is the fix you know works.
