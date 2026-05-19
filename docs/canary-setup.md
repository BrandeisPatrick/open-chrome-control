# Chrome setup for CDP control

The pattern assumes a Chrome instance launched with remote debugging enabled on port 9222 and a dedicated user-data-dir. Chrome Canary is recommended (so it doesn't fight your normal browsing session), but stable Chrome works the same way.

## Launch flags

```bash
"/Applications/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary" \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.chrome-canary-cdp-profile" \
  --no-first-run \
  --no-default-browser-check
```

Linux example:

```bash
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.chrome-cdp-profile" \
  --no-first-run \
  --no-default-browser-check
```

Windows (PowerShell):

```powershell
& "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  --remote-debugging-port=9222 `
  --user-data-dir="$env:USERPROFILE\.chrome-cdp-profile" `
  --no-first-run `
  --no-default-browser-check
```

After launch, verify:

```bash
curl -s http://127.0.0.1:9222/json/version
```

You should see a JSON blob with `Browser`, `Protocol-Version`, `webSocketDebuggerUrl`, etc. If you get nothing, the browser isn't actually listening — re-check the flags.

## Why a dedicated `--user-data-dir`

Two reasons:

1. **Session isolation.** Your CDP-controlled browser has its own cookies, history, and extensions, separate from the normal browser you use day-to-day. Sign in to the apps you want the agent to drive, and that auth lives in the CDP profile.
2. **Chrome v136+ port-block bypass.** As of Chrome 136, `--remote-debugging-port` is silently ignored when launching against the default profile. Specifying a non-default `--user-data-dir` bypasses that restriction. Without this flag, your `curl` check returns nothing and the failure mode looks like "the browser is up but CDP isn't" — confusing on first encounter.

## Profile lock

The dedicated profile can only be opened by one Chrome process at a time. If you try to launch a second instance against the same `--user-data-dir`, the new launch silently exits — no error, no window, just nothing.

If the agent reports "Chrome not running on CDP 9222" but you can see a window, check:

```bash
lsof +D "$HOME/.chrome-canary-cdp-profile" | head
ps aux | rg -i "chrome canary" | rg -v "rg -i"
```

If a previous Chrome process is still holding the lock, kill it:

```bash
pkill -f "Google Chrome Canary"
```

…then relaunch with the flags above.

## Port conflict

If `lsof -i :9222` shows a process you didn't intend to be there, choose another port (the sub-agent will fall back to 9223 and report it):

```bash
"/Applications/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary" \
  --remote-debugging-port=9223 \
  --user-data-dir="$HOME/.chrome-canary-cdp-profile"
```

You'll need to update the chrome-devtools MCP server config to match.

## Lifecycle

The browser is meant to stay running across agent sessions. Launch once, sign in to the surfaces you care about, and leave it. The next time the agent runs, it reconnects to the same instance and reuses your sessions.

Closing the window also kills the CDP listener. If you closed it accidentally, just relaunch with the same `--user-data-dir`; cookies and storage persist on disk.

## Convenience launcher

A small shell function makes this less painful:

```bash
chrome-cdp() {
  local port="${1:-9222}"
  local profile="$HOME/.chrome-canary-cdp-profile"

  if curl -s "http://127.0.0.1:${port}/json/version" >/dev/null; then
    echo "Already up on :${port}"
    return 0
  fi

  "/Applications/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary" \
    --remote-debugging-port="${port}" \
    --user-data-dir="${profile}" \
    --no-first-run \
    --no-default-browser-check \
    >/dev/null 2>&1 &

  for _ in {1..10}; do
    if curl -s "http://127.0.0.1:${port}/json/version" >/dev/null; then
      echo "Up on :${port}"
      return 0
    fi
    sleep 0.5
  done
  echo "Timed out waiting for CDP on :${port}" >&2
  return 1
}
```

Drop it in your shell rc and run `chrome-cdp` before any session that needs browser control.
