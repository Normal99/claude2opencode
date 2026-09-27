# claude2opencode

Import your **Claude Code** conversation history into **OpenCode**.

OpenCode (v2) has no native importer for Claude Code transcripts yet. `claude2opencode`
converts the JSONL transcripts that Claude Code stores under `~/.claude/projects/` into
OpenCode's session JSON format and imports them with `opencode session import`, so your
past conversations appear in the OpenCode session picker with their original titles,
timestamps, tool calls, thinking, and token/cost totals.

## Requirements

- Python 3.8+ (no third-party dependencies)
- [OpenCode](https://opencode.ai) v2 CLI (`opencode session import`)

## Install

```bash
git clone https://github.com/Normal99/claude2opencode.git
mkdir -p ~/.local/bin
ln -s "$PWD/claude2opencode/claude2opencode" ~/.local/bin/claude2opencode
```

Make sure `~/.local/bin` is on your `PATH`.

## Usage

```bash
# Show every Claude Code session found on this machine
claude2opencode list

# Convert a single transcript to OpenCode session JSON (no import)
claude2opencode convert ~/.claude/projects/-home-volkan/<uuid>.jsonl

# Convert every transcript in ~/.claude/projects/*/
claude2opencode convert-all [outdir]

# Import a converted JSON file (converts it first if needed)
claude2opencode import /tmp/opencode/import/<uuid>.import.json

# Convert + import everything
claude2opencode import-all
```

Converted files are written to `/tmp/opencode/import/` by default (pass a different
`outdir` to `convert-all`, or a directory after the file for `convert`).

## What gets converted

| Claude Code | OpenCode |
|---|---|
| User text messages | `user` messages |
| Assistant `text` blocks | `text` content |
| Assistant `thinking` blocks | `reasoning` content |
| `tool_use` blocks | `tool` content (input, name, id) |
| The following `tool_result` line | Merged into the matching tool's result; `is_error` results become error tool states |
| One JSONL line per content block with the same `message.id` | Merged into a single assistant message |
| `message.usage` | Per-message and session `tokens` |
| Public Anthropic list prices | Approximate `cost` |
| `cwd` recorded in the transcript | Session `location`, so sessions land under the right project |
| `ai-title` events, else first real user message | Session title |

**Skipped as noise:** slash-command messages (`/login`, `/model`), local-command stdout,
`<pasted_content>` marker blobs, `<local-command-caveat>` caveats, and subagent
(sidechain) transcripts.

## Continuing very long conversations

A full transcript import is an **archive**: it renders fine, but you cannot send a
message in it, because no model can ingest hundreds of thousands of tokens of history.
OpenCode's compaction can't fix that either — the summary request is itself too long.

For every transcript whose history exceeds a small token budget, the converter therefore
emits a **second file**: a compact *continuation* session (`<uuid>.cont.import.json`)
holding only the most recent tail (≈15k–22k tokens, truncated if needed). It appears in
the session picker right next to the archive as `... (continued)` — open that one to
pick up where Claude Code left off:

```
ses_3c178070-…        [ARCHIVE · read-only] Get overview and summary…      <- browse only
ses_3c178070-…-cont   [CONTINUE HERE] Get overview and summary…            <- resume here
```

The prefixes exist because the picker truncates long titles: a "(continued)"
suffix gets cut off and people resume the archive by mistake — where the model
reads millions of tokens, fails, and appears to "give up". `import-all`
imports archives first, then continuations.

## Caveats

- Cost figures are approximations based on public Anthropic list prices; edit the
  `PRICING` table in the script if you want different rates.
- Empty `thinking` blocks (signature-only) are dropped.
- Very large transcripts may hit OpenCode's own import batch limits; if a session with
  thousands of messages fails to import, split the `messages` array in the converted JSON.
- Tool calls whose result never arrived in the transcript are imported as errored tools
  rather than dropped, so the session stays readable.
- OpenCode may ship a native Claude-format importer in a future release; until then this
  is the working path.

## How it works

OpenCode V2 exposes an experimental session-transfer endpoint
(`POST /api/experimental/session/import`, wrapped by `opencode session import`) that
accepts a single JSON document:

```json
{
  "info":     { "id": "ses_...", "projectID": "...", "time": {...}, "tokens": {...},
                "cost": 0, "location": {"directory": "/abs/path"}, "title": "...", ... },
  "messages": [ { "type": "user", ... }, { "type": "assistant", "content": [...], ... } ]
}
```

The exact shape was reverse-engineered from the running server's OpenAPI spec
(`opencode api get /openapi.json`) and a real `opencode session export`, so the
converter emits precisely what the server's decoder accepts.

## License

MIT
