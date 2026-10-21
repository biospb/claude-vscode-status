# Claude VS Code Status (biospb fork)

A small, local-only VS Code extension that shows Claude Code session information in the VS Code status bar.

```text
Opus 5 (High) │ Context 4% · 38.9k/1M │ 5h 45% │ 7d 5%
```

This is a fork of [E-MRE/claude-vscode-status](https://github.com/E-MRE/claude-vscode-status). It keeps the upstream behaviour and privacy model and makes the status bar **follow the conversation you are looking at**. With several Claude conversations open in one workspace, upstream often showed the wrong one.

It does not sign in to your Anthropic account, call unofficial APIs, send telemetry, or upload your conversations anywhere.

## What this fork changes

### Follows the focused conversation

When Claude conversations are open as editor tabs, the status bar shows the conversation in the **focused tab**. Switch tabs and the status bar switches with you.

The extension reads each conversation's title from its transcript (`ai-title` lines) and matches it to the active tab's label. VS Code shortens long tab labels with `…`, so the match is by prefix.

### Better automatic session detection

The extension picks a session in this order:

| Priority | Source | Shown in tooltip as |
|---|---|---|
| 1 | Session you pinned manually | `pinned` |
| 2 | Conversation in the focused editor tab | `focused tab` |
| 3 | Live, busy Claude process for this workspace (`~/.claude/sessions/<pid>.json`) | `active session` |
| 4 | Transcript with the most recent assistant reply | `most recent` |

Fixes compared to upstream:

- **No more flicker** between two running conversations. A live, busy session is preferred.
- **Ranks by the last real assistant reply**, not by file modification time. An idle Claude process keeps touching its transcript, which made a days-old session look newest.
- **Trusts the workspace's own transcript folder** (`~/.claude/projects/<encoded-workspace>/`). Upstream filtered by the transcript's working directory, which changes whenever the agent runs `cd`, so active sessions were sometimes skipped.

### Three color levels for context size

The status item changes color as the context grows, so you know when to start a fresh conversation:

| Context tokens | Look | Setting |
|---|---|---|
| ≥ 150k | yellow text | `claudeVscodeStatus.warnTokens` |
| ≥ 200k | orange background | `claudeVscodeStatus.hotTokens` |
| ≥ 256k | red background | `claudeVscodeStatus.errorTokens` |

The levels use token counts, not percentages, so they behave the same whatever the context window size. Set any of them to `0` to turn it off.

### Smaller changes

- The session picker and tooltip show the **conversation title**. Live sessions are marked in the picker.
- The tooltip has a **Selected via** line that tells you why this session was chosen.
- The `5h` / `7d` parts are **hidden** until the usage bridge sends data, instead of showing `–%`.
- Million-token counts use an uppercase `M` (`1M`).
- New `claudeVscodeStatus.debug` setting writes session-detection details to the **Claude Code Status** output channel.

## What it shows

- Active Claude model, for example `Opus 5` or `Sonnet 5`
- Effort level, when Claude exposes it
- Context-window usage as a percentage
- Current context tokens and context-window size
- 5-hour and 7-day usage percentage (needs the optional bridge)

Hover over the status item for details: conversation title, fresh input tokens, cache-write and cache-read tokens, last output tokens, rate-limit reset times, Claude Code version, data source, the selected session, and why it was selected.

## How it works

The extension uses local files only.

### Native Claude Code for VS Code

For conversations in the official Claude Code VS Code extension, model, effort, token, and title data are read from Claude Code's transcripts:

```text
~/.claude/projects/
```

Live-session information comes from:

```text
~/.claude/sessions/
```

Only the data needed for the status display is kept in memory. Transcript contents are never sent anywhere.

### Optional 5h / 7d usage bridge

Claude's native VS Code transcript does not include account-level 5-hour and 7-day usage. To get these values, the extension can install a small local `statusLine` bridge in Claude Code. On first run, VS Code asks whether you want to enable it.

If you enable it, the extension:

1. Backs up `~/.claude/settings.json`.
2. Writes a small bridge script under `~/.claude/vscode-status/`.
3. Registers it as Claude Code's `statusLine` command.
4. Stores only the needed status data in `~/.claude/vscode-status/sessions/`.

If you already have a `statusLine`, the extension asks before replacing it and saves the old one so it can be restored.

Remove the bridge at any time with **Claude Status: Disable Usage Bridge and Restore Previous StatusLine**.

## Installation

This fork is not on the VS Code Marketplace. The Marketplace extension **Claude VS Code Status** by **E-MRE** is the upstream version, without the changes above.

### Build a VSIX

```bash
git clone https://github.com/biospb/claude-vscode-status.git
cd claude-vscode-status
npm install
npx @vscode/vsce package
```

Then in VS Code: **Extensions** → `...` menu → **Install from VSIX...** → select the generated `.vsix` file, and reload if asked.

If you have the Marketplace version installed, uninstall it first. Both use the same extension ID.

## Multiple Claude conversations

By default the status bar follows the focused conversation tab, then the busy session, then the most recent one (see the table above).

Click the status item to pick and pin a specific conversation. Run **Claude Status: Clear Session Pin (Auto-detect)** to go back to automatic selection.

## Commands

Open the Command Palette and type `Claude Status`:

- **Enable/Update 5h & 7d Usage Bridge**: installs or updates the local bridge.
- **Refresh**: updates the display now.
- **Select/Pin Session**: picks a Claude conversation for this workspace.
- **Clear Session Pin (Auto-detect)**: goes back to automatic selection.
- **Disable Usage Bridge and Restore Previous StatusLine**: removes the bridge and restores the previous `statusLine`.

## Settings

| Setting | Default | Description |
|---|---|---|
| `claudeVscodeStatus.refreshInterval` | `2` | Seconds between checks of local files. No network requests. |
| `claudeVscodeStatus.showTokens` | `true` | Show token counts next to the context percentage. |
| `claudeVscodeStatus.defaultContextWindowSize` | `1000000` | Context window size to use when the transcript does not say. The tooltip marks it as a fallback. |
| `claudeVscodeStatus.warnTokens` | `150000` | Yellow text from this many context tokens. `0` turns it off. |
| `claudeVscodeStatus.hotTokens` | `200000` | Orange background from this many context tokens. `0` turns it off. |
| `claudeVscodeStatus.errorTokens` | `256000` | Red background from this many context tokens. `0` turns it off. |
| `claudeVscodeStatus.debug` | `false` | Write session-detection details to the **Claude Code Status** output channel. |

## Platform support

- Windows: the fork is developed and tested here (PowerShell bridge).
- macOS: the main platform upstream.
- Linux: uses the same POSIX bridge as macOS.

## Privacy and security

The extension is local-only:

- No telemetry or analytics
- No network requests
- No Anthropic credentials requested or stored
- No cookies read
- No unofficial Claude API used
- No conversation data uploaded

The optional bridge receives Claude Code's `statusLine` JSON locally and writes a reduced copy to `~/.claude/vscode-status/`. Bridge cache entries older than seven days are deleted automatically.

## Compatibility note

The files under `~/.claude` (transcripts, `sessions/*.json`, `ai-title` lines) are internal to Claude Code and can change between releases. The parser is careful, but a Claude Code update may break detection until the extension is updated. Focused-tab matching also depends on how the Claude extension titles its tabs.

The 5h/7d fields depend on Claude Code including rate-limit data in its `statusLine` input.

## Troubleshooting

- **`Claude: waiting for data`**: send a message in Claude Code, wait for the reply, then run **Claude Status: Refresh**.
- **Wrong conversation shown**: hover the status item and check **Selected via**. If it isn't what you expect, turn on `claudeVscodeStatus.debug` and check the **Claude Code Status** output channel, or click the status item and pin the right session.
- **No 5h / 7d values**: enable the usage bridge from the Command Palette. Values appear after Claude Code sends status data.
- **You already use a custom `statusLine`**: read the confirmation dialog before enabling the bridge. The extension saves your configuration and restores it when the bridge is removed.

## Development

Requires Node.js, npm, and VS Code.

```bash
npm install
npm run compile   # or: npm run watch
```

Press `F5` in VS Code to start an Extension Development Host. The code is in [src/extension.ts](src/extension.ts).

## Credits

Original extension by [E-MRE](https://github.com/E-MRE/claude-vscode-status). Fork changes by [biospb](https://github.com/biospb/claude-vscode-status).

## License

MIT. See [LICENSE](LICENSE).

## Disclaimer

This is an independent community project. It is not affiliated with, endorsed by, or sponsored by Anthropic. Claude and Claude Code are trademarks of their owner.
