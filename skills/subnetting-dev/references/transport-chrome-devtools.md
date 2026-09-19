# Transport: chrome-devtools MCP

The [chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp) server attaches to the user's running Chrome with `--autoConnect`, so the agent works in the tab the user already has open, with their cookies and their signed-in account. Tool names below are bare; a harness may show them under a server namespace.

## Setup

Prerequisites: Chrome 144 or later (check `chrome://settings/help`) and Node.js 20.19 or later for `npx`.

1. Add the server to the harness.

   | Harness                                  | How                                                                                              |
   | ---------------------------------------- | ------------------------------------------------------------------------------------------------ |
   | Claude Code with the `subnetting` plugin | Nothing; the plugin bundles the server                                                           |
   | Claude Code, skill only                  | `claude mcp add chrome-devtools --scope user -- npx -y chrome-devtools-mcp@latest --autoConnect` |
   | Codex CLI                                | `codex mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect`               |
   | Anything else                            | The JSON entry below, in that harness's MCP config                                               |

   ```json
   {
     "mcpServers": {
       "chrome-devtools": {
         "command": "npx",
         "args": ["-y", "chrome-devtools-mcp@latest", "--autoConnect"]
       }
     }
   }
   ```

   A harness that already runs this server needs no second copy.

2. In Chrome, open `chrome://inspect/#remote-debugging` and turn the toggle on. The setting persists across restarts, so this happens once.
3. Open the target origin in that Chrome and sign in if the work involves cloud networks.
4. On the first tool call Chrome shows a native prompt asking to allow the connection. The user clicks **Allow**; the approval lasts until Chrome closes.

When no such tool is available to you, give the user these steps and stop; do not improvise another way into their browser.

## Intent to tool

| Intent                      | Tool                            | Notes                                                                                                                  |
| --------------------------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Call the facade             | `evaluate_script`               | `function` is a string holding a function declaration; use the async shape from [subnetting-app.md](subnetting-app.md) |
| Find and pick the tab       | `list_pages` then `select_page` | Match the target origin; prefer a network view over the docs                                                           |
| Open a fresh tab            | `new_page`                      | Only when no tab on the origin exists                                                                                  |
| Fetch the contract          | your HTTP tool                  | Or `evaluate_script` with `async () => (await fetch('/llms.txt')).text()`                                              |
| Navigate within the app     | `navigate_page`                 | Element uids invalidate afterwards                                                                                     |
| Wait for a view to settle   | `wait_for`                      | Pass the texts you expect to see                                                                                       |
| Read the accessibility tree | `take_snapshot`                 | One `uid` per element; uids invalidate on navigation or a structural DOM change                                        |
| Look at the canvas          | `take_screenshot`               | Optional `uid` for an element-only shot; the canvas is SVG, which text snapshots cannot see                            |
| Click a row or button       | `click`                         | A `uid` from the latest snapshot                                                                                       |
| Fill one field, or several  | `fill`, `fill_form`             | For CIDR changes prefer the facade, which skips the input's confirm modal                                              |
| Type at the current focus   | `type_text`                     | When no input has a uid                                                                                                |
| Press a shortcut            | `press_key`                     | Only for the keyboard-only flows listed in [subnetting-app.md](subnetting-app.md)                                      |
| Read console output         | `list_console_messages`         | Then `get_console_message` with the message id for the full text                                                       |
| Make the window wide enough | `resize_page`                   | At least 1024 px wide; narrower windows render the compact drawer layout, a different DOM                              |

## Eval pitfalls

- `evaluate_script` takes a function, not a bare expression. `window.subnet.getNetwork()` on its own fails; `() => JSON.stringify(window.subnet.getNetwork())` works.
- Return the awaited, stringified result from inside the function. Returning the promise of a facade call without awaiting it can serialize as `undefined`.
- When saving a snapshot or screenshot to a file, the path has to sit inside the workspace; paths outside it are rejected. Delete the file afterwards.
- `hover` and `click` can time out on tree rows (hover-revealed quick actions, context menus). Fall back to dispatching native events through `evaluate_script` (`mouseover`, `contextmenu`, `el.click()`); they still exercise the app's full command path.

## Alternative: an isolated Chrome profile

Attaching to the main profile exposes everything in it (see the security note). For UI QA, or whenever the user's account is not needed, drive a second Chrome on its own profile instead. It coexists with the main Chrome and starts signed out. Chrome ignores `--remote-debugging-port` on the default profile, which is why the dedicated profile directory is required.

```bash
# macOS paths; adjust the binary and the profile directory for other systems.
mkdir -p "$HOME/.cache/chrome-mcp-profile"
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --user-data-dir="$HOME/.cache/chrome-mcp-profile" \
  --remote-debugging-port=9222 --no-first-run --no-default-browser-check \
  "https://subnetting.dev" > /dev/null 2>&1 &
```

Point the server at it with `--browserUrl http://127.0.0.1:9222` in place of `--autoConnect`. When the server config is fixed on `--autoConnect`, it discovers Chrome through the `DevToolsActivePort` file of the default profile; overwriting that file with the second instance's port and browser WebSocket path (both from `http://127.0.0.1:9222/json/version`) redirects it for the session.

## Troubleshooting

- `list_pages` fails to connect: Chrome is older than 144, the remote-debugging toggle is off, or the Allow prompt was dismissed. Fix, restart Chrome if the toggle changed, and retry.
- "Target closed": Chrome closed or crashed mid-session. Reopen it on the target origin and repeat the call.
- No tab on the origin is listed: the tab lives in a different Chrome instance or profile than the one the server attached to.
- `npx` cannot find the package: check `node --version` and rerun the add command.

## Security note

A server attached to the main profile can inspect, debug and modify any browser data while connected: every open tab, every signed-in session, not only subnetting.dev. Say so plainly when the user sets this up. Keep sensitive work out of that Chrome window while the agent is active, and revoke access when done by closing Chrome or turning the `chrome://inspect/#remote-debugging` toggle off.
