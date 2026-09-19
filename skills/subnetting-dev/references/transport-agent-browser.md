# Transport: agent-browser CLI

[agent-browser](https://github.com/vercel-labs/agent-browser) is a shell CLI that drives its own headless Chrome. That browser starts signed out with empty storage, which suits UI QA and scratch networks on `/`; it does not see the user's tab or account. Tested with 0.26.0. Run `agent-browser --help` for the full command list; this file holds only what driving subnetting.dev needs.

## Bootstrap

```bash
agent-browser open https://subnetting.dev && agent-browser wait --load networkidle
agent-browser set viewport 1920 1080   # at least 1024 px wide, or the compact layout renders
curl -s https://subnetting.dev/llms.txt   # the contract; read it before choosing methods
agent-browser eval 'JSON.stringify(window.subnet.getNetwork())'
```

Swap the origin for the one you are driving.

## Calling the facade

One-line synchronous reads fit in a single-quoted `eval`. Everything else goes through `eval --stdin` with a quoted heredoc delimiter, wrapping the async function from [subnetting-app.md](subnetting-app.md) as an immediately invoked expression:

```bash
agent-browser eval --stdin <<'EVAL'
(async () => {
  const { data: net } = await window.subnet.getNetwork()
  const r = await window.subnet.split(net.rootBlockId, 4)
  return JSON.stringify(r)
})()
EVAL
```

Why the heredoc: the shell rewrites inner double quotes, backticks, `$()` and `!` before agent-browser sees a single-quoted script. The quoted `'EVAL'` delimiter passes the body through untouched.

## Intent to command

| Intent                           | Command                                                             |
| -------------------------------- | ------------------------------------------------------------------- |
| Open a page                      | `agent-browser open <url> && agent-browser wait --load networkidle` |
| Press a shortcut                 | `agent-browser press f`                                             |
| Read the accessibility tree      | `agent-browser snapshot -i` (element refs like `@e3`)               |
| Click or fill a modal field      | `agent-browser click @e3`, `agent-browser fill @e4 "text"`          |
| Look at the canvas               | `agent-browser screenshot`, with `--full` or `--annotate`           |
| What changed since last snapshot | `agent-browser diff snapshot`                                       |
| Read console output              | `agent-browser console`, `agent-browser errors`                     |

`--annotate` overlays numbered labels with a legend, which is the quickest way to reason about block positions on the SVG canvas. Refs (`@e1`, `@e2`) invalidate on navigation and on any DOM change; snapshot again after `open` or after an action that changes the page.

## Reaching a Chrome that is already running

`agent-browser --cdp 9222 <command>` attaches to a Chrome started with `--remote-debugging-port=9222`, such as the isolated profile in [transport-chrome-devtools.md](transport-chrome-devtools.md). Use it when the work needs a visible window or a signed-in session kept apart from the user's main profile.

## Cleaning up

The headless browser keeps its storage between commands in a session, so the scratch network on `/` survives until the session ends. Undo or delete test mutations when the task was QA rather than a build the user asked for.
