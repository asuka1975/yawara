# yawara

English | [日本語](README-ja.md)

Claude Code hooks that rewrite Claude's replies and the questions and options of `AskUserQuestion` into plain, easy-to-read text before they are shown, using Antigravity CLI (`agy`) with Gemini 3.8 Flash (high). Distributed as an [APM (Agent Package Manager)](https://github.com/microsoft/apm) package.

Every original and rewritten text is logged under `~/.yawara/logs/`.

## Requirements

- Claude Code with support for the `MessageDisplay` hook (tested with v2.1.310)
- APM CLI (`apm`, tested with v0.29.0)
- Antigravity CLI (`agy`, tested with v1.4.7), already signed in
  - The hooks call `agy -p` (headless mode). Start `agy` interactively once beforehand and sign in.
- `python3` 3.9 or later (standard library only)

## Installation

For every Claude Code session (user scope, registered in `~/.claude/settings.json`):

```bash
apm install -g asuka1975/yawara#v0.1.1
```

For a single project only (registered in that project's `.claude/settings.json`):

```bash
cd your-project
apm install asuka1975/yawara#v0.1.1
```

The hooks take effect in Claude Code sessions started after installation. To remove them, run `apm uninstall asuka1975/yawara` (add `-g` for a global install). The bare name, as in `apm uninstall yawara`, is not accepted.

## What it does

| Target | Hook | Behavior |
| --- | --- | --- |
| Claude's reply messages | `MessageDisplay` | Holds back the lines while the message streams in, then rewrites the whole message once it is complete and displays the result. |
| `AskUserQuestion` questions | `PreToolUse` (matcher: `AskUserQuestion`) | Rewrites the question text (`question`), option names (`label`), and option descriptions (`description`) in a single call and swaps them in through `updatedInput`. `header` and previews are left unchanged. |

Rewriting rules (the prompts live in `.apm/hooks/scripts/yawara.py`):

- Write in the same language as the original. Keep the meaning, facts, and conclusions; do not drop or add information.
- Put the conclusion first and split long sentences. Keep technical terms, adding a short plain-language gloss in parentheses where helpful.
- Leave code, commands, file paths, URLs, numbers, identifiers, and error messages exactly as they are.

The prompts themselves are written in Japanese, but the rewritten text stays in the language of the original message.

### Things to know

- **Display is delayed.** In our measurements a single rewrite took about 10–70 seconds, and the message is not shown in the meantime (the spinner reads "yawara: rewriting the message for readability…"). The rewritten message may appear only after the turn is marked "done".
- **Only the on-screen text changes.** The transcript and the text Claude itself sees keep the original wording. The verbose view (`ctrl+o`) also shows the original.
- **`AskUserQuestion` answers reach Claude in the rewritten wording.** The hook therefore passes Claude a mapping back to the original wording through `additionalContext`.
- **Failures never block Claude.** If `agy` errors out, times out, or returns malformed output, the original text is shown and the log records `"status": "fallback"` with the reason. Messages larger than 100 KB are not rewritten, because the prompt is passed to `agy` as a command-line argument.
- **Displayed text is sent to Google (Gemini).** Everything in Claude's replies, including code and file paths, is passed to `agy` as is. For projects whose content must not leave your machine, install per project only, or turn the hooks off with `YAWARA_DISABLE=1`.

## Logs

`~/.yawara/` is created automatically the first time a hook runs.

```text
~/.yawara/
├── logs/YYYY-MM-DD.jsonl   # Before/after log (one record per line, one file per day)
├── state/                  # Temporary buffer for messages still streaming (deleted once shown)
└── workspace/              # Working directory for agy (keeps project context out of the rewrite)
```

Shape of a log record:

```json
{
  "timestamp": "2026-10-26T02:23:12+09:00",
  "kind": "message",
  "status": "rewritten",
  "error": null,
  "model": "gemini-3.8-flash-high",
  "duration_sec": 24.8,
  "session_id": "…",
  "cwd": "/path/to/project",
  "message_id": "…",
  "before": "original text",
  "after": "rewritten text"
}
```

- `kind`: `message` (a reply) or `ask_user_question` (a question). For `ask_user_question`, `tool_use_id` replaces `message_id`, and `before` / `after` hold the `questions` array.
- `status`: `rewritten`, or `fallback` (rewriting failed and the original text was shown; `after` is `null`).

Example: print today's log in a readable form

```bash
jq -r '"[\(.timestamp)] \(.kind) \(.status)\n--- before\n\(.before)\n--- after\n\(.after)\n"' ~/.yawara/logs/$(date +%F).jsonl
```

## Configuration

Configure the hooks with environment variables. To pass them to Claude Code, put them under `env` in `~/.claude/settings.json`, or set them in the shell that launches Claude Code.

| Variable | Default | Description |
| --- | --- | --- |
| `YAWARA_MODEL` | `gemini-3.8-flash-high` | Model passed to `agy`. Run `agy models` to list the available models. |
| `YAWARA_TIMEOUT` | `150` | Time limit for one `agy` call, in seconds. Keep it below the hook's own `timeout` (180 seconds); otherwise the hook is cut off first and only part of the original text is shown. |
| `YAWARA_HOME` | `~/.yawara` | Where logs and temporary files are stored. |
| `YAWARA_AGY` | `agy` | Path to the `agy` command. If `agy` is not on the `PATH` of Claude Code (for example when launched from the desktop app), every log record becomes a `fallback` with `FileNotFoundError`. In that case, set this to the absolute path printed by `which agy`. |
| `YAWARA_DISABLE` | (unset) | Set to `1` to skip rewriting and show the original text. |

Example (`~/.claude/settings.json`):

```json
{
  "env": {
    "YAWARA_MODEL": "gemini-3.8-flash-medium"
  }
}
```

## Package layout

```text
yawara/
├── apm.yml                      # APM manifest (targets: claude)
└── .apm/hooks/
    ├── yawara.json              # Hook definitions (MessageDisplay / PreToolUse)
    └── scripts/yawara.py        # Rewriting and logging
```

`apm install` merges the hook definitions into `.claude/settings.json` (`~/.claude/settings.json` for a global install) and places the script at `.claude/hooks/yawara/scripts/yawara.py`.

## Trying it out

The hook reads JSON from standard input, so you can try it without Claude Code:

```bash
echo '{"hook_event_name":"MessageDisplay","message_id":"test","index":0,"final":true,"delta":"Returning updatedInput from a PreToolUse hook replaces the tool input.\n"}' \
  | YAWARA_HOME=/tmp/yawara-test python3 .apm/hooks/scripts/yawara.py
```

## License

[GNU General Public License v3.0 or later](LICENSE) (`GPL-3.0-or-later`)
