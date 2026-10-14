# Claude Code + OpenRouter (DeepSeek, Kimi, GLM, Qwen…)

Run the Claude Code CLI against any model on [OpenRouter](https://openrouter.ai) through its
Anthropic-compatible endpoint, using short commands that you define yourself, for example
`ccds` for DeepSeek V4 Pro.

- **You choose the commands.** Each one is an alias in
  [`aliases.json`](#aliases-config-file-aliasesjson) that maps a name to an OpenRouter model. Add,
  change or remove them with `claude-or --add` / `--update` / `--remove` or by editing the file.
  You never need to touch the script.
- The environment variables apply only to that one `claude` process. Plain `claude` keeps using
  your normal Anthropic / claude.ai login.
- The commands are real executables in `PATH`, not shell aliases, so they also work from scripts
  and when a running Claude Code session calls them via its Bash tool.
- Runs on macOS or Linux with `zsh`, `curl`, [`jq`](#requirements-jq) and Claude Code. On Linux
  there is no Keychain, so supply the key through the env var or the key file
  (see [API key](#api-key)).

## Contents

- [Requirements: `jq`](#requirements-jq)
- [Commands](#commands)
- [Install](#install)
- [Files](#files)
- [API key](#api-key)
- [Aliases config file (`aliases.json`)](#aliases-config-file-aliasesjson)
- [Add a new alias](#add-a-new-alias)
  - [With `--add` (recommended)](#with---add-recommended)
  - [By hand](#by-hand)
  - [Other options](#other-options)
- [Change an alias](#change-an-alias)
- [Remove an alias](#remove-an-alias)
- [Use only `claude-or`, without alias commands](#use-only-claude-or-without-alias-commands)
- [Delegating from a Claude Code session](#delegating-from-a-claude-code-session)
- [Expected warnings](#expected-warnings)
- [What it sets](#what-it-sets)

## Requirements: `jq`

Aliases (`aliases.json`, `--add`, `--update`, `--remove`, `--list`, `--link`) need [`jq`](https://jqlang.org).

- **macOS 15+**: built in (`/usr/bin/jq`).
- **Older macOS**: `brew install jq`
- **Linux**: `sudo apt install jq` (or `dnf` / `pacman` / `apk install jq`)

Check with `jq --version`. Without `jq`, alias commands stop with an install hint, but you can still
run any model directly. That path doesn't need `jq`, and neither does `--unlink`:

```bash
claude-or deepseek/deepseek-v4-pro
```

## Commands

Default aliases from `aliases.json`:

| Command | OpenRouter model | Context |
|---|---|---|
| `ccds` | `deepseek/deepseek-v4-pro` | 1M |
| `ccdsf` | `deepseek/deepseek-v4-flash` | 1M |
| `cckimi` | `moonshotai/kimi-k3` | 1M |
| `ccglm` | `z-ai/glm-5.3` | 1M |
| `ccqwen` | `qwen/qwen3-coder-next` | 262k |
| `ccfree` | `openrouter/free` (OpenRouter's free-model router) | 200k |

Every command passes the rest of its arguments straight to `claude`, for example
`ccds -p "explain this repo"` or `ccds --resume`.

`claude-or` itself:

| Command | What it does |
|---|---|
| `claude-or <alias> [args]` | Run an alias without needing its symlink |
| `claude-or <model-id> [args]` | Run any OpenRouter model once, no alias needed (model ids are always `vendor/model`) |
| `claude-or --list` | Show aliases: name, model, context, background model |
| `claude-or --link` | Sync symlinks with `aliases.json` (create new, remove deleted) |
| `claude-or --unlink` | Remove all alias symlinks, keeping `aliases.json` |
| `claude-or --add <alias> <model-id> [context]` | Validate and add an alias to `aliases.json`, then `--link` |
| `claude-or --update <alias> <property> <value>` | Validate and set an alias's `model`, `small` or `context` |
| `claude-or --remove <alias>...` | Remove alias(es) from `aliases.json` and their symlinks |
| `claude-or --self-update` | Replace the installed script with the latest release tag from GitHub, keeping `aliases.json` |
| `claude-or --set-key` | Verify an OpenRouter key and save it to `api_key` (hidden prompt or pipe; asks before replacing an existing key) |
| `claude-or --unset-key` | Delete the `api_key` file |
| `claude-or`, `claude-or --help` | Help with all commands, examples and your current aliases |

## Install

```bash
R=https://raw.githubusercontent.com/dariusz-przada/claude-or/main
mkdir -p ~/.local/bin ~/.config/claude-or
curl -fsSL "$R/claude-or" -o ~/.local/bin/claude-or && chmod +x ~/.local/bin/claude-or
[ -f ~/.config/claude-or/aliases.json ] || curl -fsSL "$R/aliases.json" -o ~/.config/claude-or/aliases.json
~/.local/bin/claude-or --link
```

Make sure `~/.local/bin` is in your `PATH`. To update, run `claude-or --self-update` (or the same
commands again): it replaces the script with the latest release tag but keeps your own `aliases.json`.

Install by copying the script, as above, not by symlinking it from a clone of this repo.
`claude-or` creates the alias symlinks next to its real path, so with a symlinked install they
would end up in the clone instead of `~/.local/bin`.

## Files

Everything lives in `~/.config/claude-or/` (or `$XDG_CONFIG_HOME/claude-or/`):

| File | Purpose |
|---|---|
| `aliases.json` | Alias → model definitions |
| `api_key` | OpenRouter API key (optional, see below) |

## API key

The script looks for the key in this order:

1. **macOS Keychain** (recommended). This command prompts for the key, so it never lands in
   shell history:
   ```bash
   security add-generic-password -a "$USER" -s openrouter-api-key -w
   ```
2. **`$OPENROUTER_API_KEY`** environment variable.
3. **Key file** `~/.config/claude-or/api_key`:
   ```bash
   claude-or --set-key                  # prompts for the key (hidden input)
   pbpaste | claude-or --set-key        # or pipe it in, e.g. from the clipboard
   ```
   `--set-key` checks the key against OpenRouter (a rejected key isn't saved; if OpenRouter can't
   be reached it saves with a warning), writes the file with mode 600, and asks before replacing a
   key that's already there. The key is never a command argument, so it stays out of shell history.
   `claude-or --unset-key` deletes the file. Both tell you if a Keychain entry or
   `$OPENROUTER_API_KEY` is set, since those take precedence over the file.

   By hand:
   ```bash
   cat > ~/.config/claude-or/api_key   # paste key, Enter, Ctrl+D
   chmod 600 ~/.config/claude-or/api_key
   ```

GUI apps such as the Claude desktop app may not see variables exported in `~/.zshrc`. If you want
a Claude Code session to call these commands, use the Keychain or the key file.

## Aliases config file (`aliases.json`)

```json
{
  "defaults": {
    "small": "deepseek/deepseek-v4-flash"
  },
  "aliases": {
    "ccds":     { "model": "deepseek/deepseek-v4-pro", "context": 1048576 },
    "ccfree":   { "model": "openrouter/free", "context": 200000, "small": "openrouter/free" }
  }
}
```

| Field | Required | Meaning |
|---|---|---|
| `aliases.<name>` | – | Command name. It also becomes the symlink name. |
| `model` | yes | OpenRouter model id |
| `context` | no | Context window in tokens. Without it, Claude Code assumes 200k for unknown models. |
| `small` | no | Model for Claude Code's "haiku"/background calls. Falls back to `defaults.small`, then to `model`. |
| `defaults.small` | no | Background model for every alias that doesn't set its own |

Environment overrides for a single run:

| Variable | Overrides |
|---|---|
| `OR_CONTEXT` | `context` |
| `OR_SMALL_MODEL` | `small` |

## Add a new alias

To add your own alias, add an entry to `~/.config/claude-or/aliases.json` and create a symlink with
that name. The script itself never needs editing. `claude-or --add` does both in one step.

### With `--add` (recommended)

```bash
claude-or --add ccluna openai/gpt-6-luna            # context taken from OpenRouter
claude-or --add ccluna openai/gpt-6-luna 500000     # or set it yourself
```

Before writing anything, `--add` checks that:

- the alias name is valid (letters, digits, `.`, `_`, `-`) and not already in `aliases.json`;
- the name doesn't clash with an existing command in `PATH`, such as `ls` or `claude`;
- the model exists on OpenRouter. It warns if the model doesn't list tool support. If OpenRouter
  can't be reached, it adds the alias anyway with a warning.

Then it writes the entry, filling in `context` from OpenRouter if you didn't give one, and runs
`--link` to create the symlink. If `aliases.json` doesn't exist yet, `--add` creates it.

To find model ids and context windows, browse [openrouter.ai/models](https://openrouter.ai/models)
or query the API:

```bash
curl -s https://openrouter.ai/api/v1/models | jq -r '.data[] | select(.id | test("luna")) | "\(.id)  \(.context_length)"'
```

### By hand

Use this route to write a whole entry at once. To set `small` or change a single field on an
existing alias, [`--update`](#change-an-alias) is simpler.

1. **Add an entry** to the `aliases` object in `~/.config/claude-or/aliases.json`. Mind the comma
   after the previous entry:
   ```json
   "ccluna": { "model": "openai/gpt-6-luna", "context": 1050000, "small": "openai/gpt-6-luna" }
   ```

2. **Create the symlink.** Either sync all of them:
   ```bash
   claude-or --link
   ```
   or add just this one:
   ```bash
   ln -s claude-or ~/.local/bin/ccluna
   ```
   `--link` refuses a name that already exists as another command, so an alias can't shadow a
   real tool. If you create the symlink yourself, check first with `command -v ccluna`.

3. **Test it:**
   ```bash
   claude-or --list
   ccluna -p "say hi" < /dev/null
   ```

### Other options

To use an alias without a symlink, run `claude-or ccluna`. To try a model once without any alias,
run `claude-or <model-id>` (put `OR_CONTEXT=<tokens>` in front if its window isn't 200k).

## Change an alias

```bash
claude-or --update ccfree small openai/gpt-5-nano   # cheaper background model
claude-or --update ccluna context 500000            # smaller context window
claude-or --update ccqwen model qwen/qwen3-coder    # point the alias at another model
```

Before writing anything, `--update` checks that:

- the alias exists in `aliases.json`;
- the property is `model`, `small` or `context`;
- for `model` and `small`, the model exists on OpenRouter (with the same tool-support warning and
  offline fallback as `--add`); for `context`, the value is a positive whole number of tokens.

When you change `model`, the stored `context` still belongs to the old model. If the new model's
window on OpenRouter differs, `--update` prints the command that sets it.

The alias name doesn't change, so its symlink stays as it is. To rename an alias, `--add` the new
name and `--remove` the old one.

## Remove an alias

```bash
claude-or --remove ccluna          # one
claude-or --remove ccluna ccqwen   # several
```

It first checks that every name exists, so a typo leaves `aliases.json` untouched. Then it removes
the entries and runs `--link`, which deletes the matching symlinks.

By hand: delete the entry from `aliases.json` and run `claude-or --link`.

`--add`, `--update` and `--remove` rewrite `aliases.json` with `jq`, so one-line entries come back expanded
to several lines. The content stays the same.

## Use only `claude-or`, without alias commands

If you'd rather not have `ccds` and the others in your `PATH`:

```bash
claude-or --unlink
```

It deletes every symlink that points to `claude-or` and leaves `aliases.json` alone. Only
`claude-or` remains as a command, and you pass the alias or model each time:

```bash
claude-or ccds                     # alias from aliases.json
claude-or deepseek/deepseek-v4-pro # or the model id
```

`claude-or --link` brings the symlinks back. `--add` and `--remove` also run `--link`, so they
recreate all the symlinks. After either one, run `--unlink` again if you want to stay
symlink-free.

## Delegating from a Claude Code session

You can ask your main Claude Code session to hand off a sub-task, for example: *"Ask `ccds` for a
second-opinion review of `src/foo.ts`"*. The session should call it headless:

```bash
ccds -p "<self-contained prompt>" < /dev/null
```

- Use `-p`. The interactive UI can't be driven from a tool call.
- Use `< /dev/null`. Without it, `claude -p` can wait forever on an open stdin.
- In `-p` mode, permission prompts are denied. If the model needs to edit files, add
  `--permission-mode acceptEdits` or `--allowedTools ...`.
- The delegated model doesn't see your conversation, so the prompt must be self-contained.

## Expected warnings

- *"claude.ai connectors are disabled because … another auth source is set"*: expected. That run
  authenticates with the OpenRouter token.
- *"… isn't described by this version's model catalog"*: Claude Code doesn't know non-Claude
  models. Set `context` for the alias, so auto-compact uses the real window.
- In `/model`, every tier (Fable, Opus, Sonnet, Haiku) shows the alias's OpenRouter model. This
  is deliberate: a tier left unset keeps its Claude default, and picking it would send the request
  to OpenRouter's Anthropic models at full price (Fable costs $10/$50 per M tokens).
- Non-Claude models can be weaker at Claude Code's tool use (edits, subagents). Try a small task
  first.

## What it sets

```
ANTHROPIC_BASE_URL=https://openrouter.ai/api
ANTHROPIC_AUTH_TOKEN=<OpenRouter key>
ANTHROPIC_API_KEY=""                      # must be empty
ANTHROPIC_MODEL / _DEFAULT_FABLE_MODEL / _DEFAULT_OPUS_MODEL / _DEFAULT_SONNET_MODEL
  / CLAUDE_CODE_SUBAGENT_MODEL = model
ANTHROPIC_DEFAULT_HAIKU_MODEL / ANTHROPIC_SMALL_FAST_MODEL = small
ANTHROPIC_DEFAULT_*_MODEL_DESCRIPTION = "OpenRouter via claude-or"   # labels in /model
CLAUDE_CODE_MAX_CONTEXT_TOKENS = context  # only when set
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
CLAUDECODE unset                          # allows launching from inside a Claude Code session
```
