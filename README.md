# subnetting

Enables AI agents to drive [subnetting.dev](https://subnetting.dev), the keyboard-first IP network planner, in your own browser tab. No paid add-on and no server in between: your agent evaluates JavaScript in the tab, and the app's `window.subnet` facade does the rest. The method contract lives at [subnetting.dev/llms.txt](https://subnetting.dev/llms.txt) and is fetched fresh every session, so an old install keeps working.

## Install

| You use              | Install                                                                                                                      | Browser access                                                                                                                 |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Claude Code + Chrome | `/plugin marketplace add masgustavos/subnetting`, then `/plugin install subnetting@subnetting`                               | The plugin bundles chrome-devtools-mcp: turn on `chrome://inspect/#remote-debugging` once, then click Allow on the first call |
| Codex CLI            | `npx skills add masgustavos/subnetting`                                                                                      | `codex mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect`                                             |
| Other harnesses      | The same `npx skills add` (installs to `~/.agents/skills/`, read by Gemini CLI, Cursor, VS Code Copilot, opencode, Goose, Amp) | The same MCP server in that harness's config                                                                                   |

Already running chrome-devtools-mcp in Claude Code? Install the skill alone, without a second copy of the server:

```shell
npx skills add masgustavos/subnetting -a claude-code
```

Updates: Claude Code does not auto-update third-party marketplaces by default. Run `/plugin marketplace update subnetting`, or turn auto-update on for it in `/plugin` under Marketplaces. For the other installs, `npx skills update`. The app tells your agent when the installed skill is behind.

Chrome 144 or later and Node.js 20.19 or later are required for the chrome-devtools-mcp route. Any other tool that can evaluate JavaScript in the page works too; the skill explains the contract.

## Quick example

With a [subnetting.dev](https://subnetting.dev) tab open:

> Build a `/16` VPC with three AZs, four subnets each, categorized
> public/private/database/intra.

The agent finds your tab, pulls `/llms.txt`, agrees the plan with you, builds it through `window.subnet`, and audits the result, selecting and fitting the canvas as it goes so you can watch it land.

## The three gates

The agent works on your real tab and, on a cloud network, your real account. The skill makes it stop three times:

1. **Design**: it confirms the CIDR plan, names and sizes with you before the first structural change.
2. **Destructive operations**: it asks before each delete, wipe, mode switch or root CIDR change, because undo does not survive a refresh and cloud networks sync live.
3. **Verification**: it audits every block against the agreed plan before saying it is done.

## Security

Attaching chrome-devtools-mcp to your main Chrome profile lets the agent see and change anything in that browser while connected, not only subnetting.dev. The skill's `references/transport-chrome-devtools.md` shows how to use an isolated Chrome profile instead, and how to revoke access.

## Layout

```
.claude-plugin/plugin.json       plugin manifest, bundles the chrome-devtools MCP server
.claude-plugin/marketplace.json  single-plugin marketplace (source "./")
skills/subnetting-dev/           the skill; synced from the app's repository, never edited here
```

## License

MIT, see [LICENSE](LICENSE).
