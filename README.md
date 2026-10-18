# claude-or

Run Claude Code on macOS and Linux with models available through [OpenRouter](https://openrouter.ai).

## Why this project exists

Claude Code normally uses the Anthropic or claude.ai login configured on your machine. `claude-or`
starts one Claude Code process against an OpenRouter model (DeepSeek, Kimi, GLM, Qwen…) through
its Anthropic-compatible endpoint, without changing your normal Claude Code setup.

The workflow looks like this:

```bash
ccds -p "Explain this repository"
ccqwen --resume
claude-or --list
```

- **You choose the commands.** Each one is an alias in
  [`aliases.json`](#aliases-config-file-aliasesjson) that maps a short name to an OpenRouter model.
  Add, change or remove them with `claude-or --add` / `--update` / `--remove` or by editing the
  file. You never need to touch the script.
- The environment variables apply only to that one `claude` process. Plain `claude` keeps using
  your normal Anthropic / claude.ai login.
- The commands are real executables in `PATH`, not shell aliases, so they also work from scripts
  and when a running Claude Code session calls them via its Bash tool.
- Runs on macOS or Linux with `zsh`, `curl`, [`jq`](#jq) and [Claude Code](#claude-code-cli). On Linux
  there is no Keychain, so save the key with `claude-or --key-add` or use the env var
  (see [API key](#api-key)). On Windows, see [win-claude-or](#windows-and-powershell).

## Contents

- [Why this project exists](#why-this-project-exists)
- [Requirements](#requirements)
  - [Claude Code CLI](#claude-code-cli)
  - [`jq`](#jq)
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
- [Windows and PowerShell](#windows-and-powershell)

## Requirements

### Claude Code CLI

Required. `claude-or` and every alias start the `claude` command, so the Claude Code CLI must be
installed and in `PATH`. Follow the official
[installation guide](https://code.claude.com/docs/en/setup).

Check with `claude --version`.

### `jq`

Aliases (`aliases.json`, `--add`, `--update`, `--remove`, `--list`, `--sync`, `--link`, `--unlink`) and the
[key manager](#api-key) (`--key-*`) need [`jq`](https://jqlang.org).

- **macOS 15+**: built in (`/usr/bin/jq`).
- **Older macOS**: `brew install jq`
- **Linux**: `sudo apt install jq` (or `dnf` / `pacman` / `apk install jq`)

Check with `jq --version`. Without `jq`, alias commands stop with an install hint, but you can still
run any model directly. That path doesn't need `jq`:

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
| **Run** | |
| `claude-or <alias> [args]` | Run an alias without needing its symlink |
| `claude-or <model-id> [args]` | Run any OpenRouter model once, no alias needed (model ids are always `vendor/model`) |
| **Aliases** | |
| `claude-or --list` | Show aliases: name, model, context, background model, link (`on`/`off`) |
| `claude-or --add <alias> <model-id> [context]` | Validate and add an alias to `aliases.json`, then link it |
| `claude-or --update <alias> <property> <value>` | Validate and set an alias's `model`, `small` or `context` |
| `claude-or --remove <alias>...` | Remove alias(es) from `aliases.json` and their symlinks |
| `claude-or --sync` | Link every alias as `aliases.json` says (turned-off ones stay unlinked), remove symlinks of deleted aliases, and show the status of each alias, with the reason when it can't be linked |
| `claude-or --link [-y] [alias...]` | Turn the named aliases on and link them. With no names: all of them, after a `[y/N]` prompt (`-y` skips it) |
| `claude-or --unlink [-y] [alias...]` | Turn the named aliases off and remove their symlinks; `aliases.json` keeps them, so `claude-or <alias>` still works. With no names: all of them, after a `[y/N]` prompt |
| **OpenRouter keys** | |
| `claude-or --key-add <name> [key]` | Verify an OpenRouter key and save it under a name; without `[key]` it's read from a hidden prompt or a pipe |
| `claude-or --key-remove <index\|name>...` | Delete saved key(s) |
| `claude-or --key-list` | Show saved keys: active (`*`), index, name, masked key |
| `claude-or --key-switch <index\|name>` | Make a saved key the active one |
| `claude-or --key-run <index\|name> <alias\|model-id> [claude args...]` | Run an alias or model with a saved key, just this once; the active key stays as it is |
| `claude-or --key-stats [index\|name\|--all]` | Spend OpenRouter recorded for a saved key (today, week, month, total) and its limit; default: the active key. `--all` (`-a`): every saved key in one table, with a sum |
| **claude-or itself** | |
| `claude-or --self-update` | Replace the installed script with the latest release tag from GitHub, keeping `aliases.json` |
| `claude-or --self-factory-reset [-y]` | Delete `keys.json`, `aliases.json` and all alias symlinks, after a `[y/N]` confirmation (`-y`/`--yes` skips it), then restore the default `aliases.json` from GitHub and link it. The script itself, a Keychain entry and `$OPENROUTER_API_KEY` stay |
| `claude-or --self-uninstall [-y]` | Remove claude-or completely: all alias symlinks, the config dir `~/.config/claude-or/` (aliases, keys) and the `claude-or` script itself, after a `[y/N]` confirmation (`-y`/`--yes` skips it). A Keychain entry and `$OPENROUTER_API_KEY` stay |
| `claude-or`, `claude-or --help` | Help with all commands, examples and your current aliases |

## Install

```bash
R=https://raw.githubusercontent.com/dariusz-przada/claude-or/main
mkdir -p ~/.local/bin ~/.config/claude-or
curl -fsSL "$R/claude-or" -o ~/.local/bin/claude-or && chmod +x ~/.local/bin/claude-or
[ -f ~/.config/claude-or/aliases.json ] || curl -fsSL "$R/aliases.json" -o ~/.config/claude-or/aliases.json
~/.local/bin/claude-or --sync
```

Make sure `~/.local/bin` is in your `PATH`. To update, run `claude-or --self-update` (or the same
commands again): it replaces the script with the latest release tag but keeps your own `aliases.json`.

Install by copying the script, as above, not by symlinking it from a clone of this repo.
`claude-or` creates the alias symlinks next to its real path, so with a symlinked install they
would end up in the clone instead of `~/.local/bin`.

To start over, run `claude-or --self-factory-reset`: it deletes `keys.json`, `aliases.json` and the
alias symlinks, then downloads the default `aliases.json` and links it again. The script stays. To
remove everything, including the script itself, run `claude-or --self-uninstall`. Both list what
they'll delete and ask first; add `-y` (or `--yes`) to skip the question, e.g. in a script.

## Files

Everything lives in `~/.config/claude-or/` (or `$XDG_CONFIG_HOME/claude-or/`):

| File | Purpose |
|---|---|
| `aliases.json` | Alias → model definitions |
| `keys.json` | Saved OpenRouter keys and which one is active (mode 600, see [API key](#api-key)) |

## API key

Save one or more OpenRouter keys under names and pick which one is active:

```bash
claude-or --key-add work                   # prompts for the key (hidden input)
pbpaste | claude-or --key-add personal     # or pipe it in, e.g. from the clipboard
claude-or --key-add ci sk-or-v1-...        # or as an argument (ends up in shell history)
claude-or --key-list
# ┌────────┬───┬──────────┬──────────────┐
# │ ACTIVE │ # │ NAME     │ KEY          │
# ├────────┼───┼──────────┼──────────────┤
# │ *      │ 1 │ work     │ sk-or...123c │
# │        │ 2 │ personal │ sk-or...9f0a │
# │        │ 3 │ ci       │ sk-or...77ab │
# └────────┴───┴──────────┴──────────────┘
claude-or --key-switch 2                   # by index...
claude-or --key-switch personal            # ...or by name
claude-or --key-remove ci                  # also by index or name, several at once
# One run with another saved key; the active key stays unchanged.
claude-or --key-run personal ccds          # through claude-or: an alias...
claude-or --key-run 1 deepseek/deepseek-v4-flash -p "hi"   # ...or any model id, by key index
OR_KEY=personal claude-or ccds             # the same with the OR_KEY variable
OR_KEY=personal ccds                       # through an alias symlink (only OR_KEY works here)
OR_KEY=2 ccdsf --resume                    # with Claude Code args, key by index
claude-or --key-stats                      # spend of the active key, or: --key-stats 2, --key-stats work
# ┌────────────┬───────────────────────────────────────────────────────────┐
# │ key        │ work (sk-or...123c), active                               │
# ├────────────┼───────────────────────────────────────────────────────────┤
# │ today      │ $0.42                                                     │
# │ this week  │ $3.10                                                     │
# │ this month │ $12.80                                                    │
# │ total      │ $57.33                                                    │
# │ limit      │ $100.00 monthly, $87.20 left, resets 2026-11-01 00:00 UTC │
# └────────────┴───────────────────────────────────────────────────────────┘
claude-or --key-stats --all                # every saved key, plus the sum
# ┌────────────┬───────┬───────────┬────────────┬────────┬───────────────────────────┐
# │ KEY        │ TODAY │ THIS WEEK │ THIS MONTH │ TOTAL  │ LIMIT                     │
# ├────────────┼───────┼───────────┼────────────┼────────┼───────────────────────────┤
# │ * work     │ $0.42 │ $3.10     │ $12.80     │ $57.33 │ none                      │
# │   personal │ $0.00 │ $0.02     │ $0.02      │ $0.02  │ $10.00 weekly, $9.98 left │
# │   ci       │ -     │ -         │ -          │ -      │ rejected by OpenRouter    │
# │ all keys   │ $0.42 │ $3.12     │ $12.82     │ $57.35 │                           │
# └────────────┴───────┴───────────┴────────────┴────────┴───────────────────────────┘
```

- Keys live in `~/.config/claude-or/keys.json` (mode 600) together with the name of the active key.
- `--key-add` checks the key against OpenRouter: a rejected key isn't saved; if OpenRouter can't be
  reached it saves with a warning. The first key you add becomes active.
- Names must be unique and use letters, digits, `.`, `_`, `-`, and can't be only digits, so a number
  always means an index. Indexes are positions in the list (1, 2, 3, …): removing a key moves the
  ones after it up.
- `--key-stats` asks OpenRouter (`GET /api/v1/key`) what it billed to that key (`--all`: to each saved
  key; a key OpenRouter rejects shows the error in its row and is left out of the sum); weeks and months are
  UTC, weeks start on Monday. It shows cost only: OpenRouter doesn't report tokens per key, and
  claude-or never sees Claude Code's API traffic. BYOK spend and a separate budget appear when set.
- Removing the active key leaves no key active until you `--key-switch` to another one.
- The `api_key` file and `--set-key` / `--unset-key` from older versions are no longer used. If you
  had a key there, add it again with `claude-or --key-add <name>`.

The script looks for the key in this order:

1. **The key given with `--key-run`**, or else with `OR_KEY=<index|name>`, for that run only.
   `OR_KEY` also works with alias symlinks: `OR_KEY=work ccds`.
2. **`$OPENROUTER_API_KEY`** environment variable, e.g. for a one-off run with a key that isn't saved.
3. **The active key** in `keys.json`.
4. **macOS Keychain**. This command prompts for the key, so it never lands in shell history:
   ```bash
   security add-generic-password -a "$USER" -s openrouter-api-key -w
   ```

GUI apps such as the Claude desktop app may not see variables exported in `~/.zshrc`. If you want
a Claude Code session to call these commands, use saved keys (`--key-add`) or the Keychain.

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
| `link` | no | `false` when the alias was turned off with `--unlink`: no symlink, but `claude-or <alias>` works. Removed by `--link <alias>`. |
| `defaults.small` | no | Background model for every alias that doesn't set its own |

Environment overrides for a single run:

| Variable | Overrides |
|---|---|
| `OR_CONTEXT` | `context` |
| `OR_SMALL_MODEL` | `small` |
| `OR_KEY` | the key: a saved key's index or name, like `--key-run`, but it also works with alias symlinks (`OR_KEY=work ccds`) |

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
- the name doesn't clash with an existing command in `PATH` (such as `ls` or `claude`), a shell
  builtin or keyword (such as `cd` or `echo`), or a file that's already next to the script;
- the model exists on OpenRouter. It warns if the model doesn't list tool support. If OpenRouter
  can't be reached, it adds the alias anyway with a warning.

Then it writes the entry, filling in `context` from OpenRouter if you didn't give one, and
links it. If `aliases.json` doesn't exist yet, `--add` creates it.

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

2. **Create the symlink.** Either link just this alias, or sync all of them:
   ```bash
   claude-or --link ccluna
   claude-or --sync
   ```
   or add just this one:
   ```bash
   ln -s claude-or ~/.local/bin/ccluna
   ```
   `--link` and `--sync` refuse a name that already exists as another command, so an alias can't shadow a
   real tool. If you create the symlink yourself, check first with `command -v ccluna`.

   `--sync` (and `--link`) show a table with one row per alias:
   ```
   ┌──────────┬────────┬────────────────────────────────────────────────────────────────┐
   │ STATUS   │ ALIAS  │ DETAILS                                                        │
   ├──────────┼────────┼────────────────────────────────────────────────────────────────┤
   │ linked   │ ccluna │ ~/.local/bin/ccluna                                            │
   │ ok       │ ccds   │ already linked                                                 │
   │ off      │ ccfree │ turned off with --unlink; claude-or --link ccfree to turn on   │
   │ skipped  │ git    │ another 'git' is already installed: /usr/bin/git               │
   │ unlinked │ ccx    │ another 'ccx' is already installed: /opt/homebrew/bin/ccx      │
   │ removed  │ ccold  │ no longer in aliases.json                                      │
   └──────────┴────────┴────────────────────────────────────────────────────────────────┘
   ```
   - `skipped`: no symlink, because of the reason shown: another command of that name in `PATH`,
     a shell builtin, an invalid name, or a file that isn't a claude-or symlink.
   - `unlinked`: the alias was linked, but a command with the same name was installed later. The
     old symlink is removed so it doesn't hide that command.
   - `off`: turned off with `--unlink`, so `--sync`, `--add`, `--remove` and `--self-update` don't
     link it.
   - Aliases without a symlink still run as `claude-or <alias>`. `--sync` exits with status 1 when
     there are any, and warns if the symlinks' directory isn't in your `PATH`.
   - Only symlinks that point to `claude-or` are ever created or removed. Files and links that
     belong to anything else are left alone.

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

It first checks that every name exists in `aliases.json`, so a typo leaves it untouched and a
name that isn't an alias (`claude-or --remove ls`) is refused. Then it removes
the entries and deletes the matching symlinks.

By hand: delete the entry from `aliases.json` and run `claude-or --sync`.

`--add`, `--update` and `--remove` rewrite `aliases.json` with `jq`, so one-line entries come back expanded
to several lines. The content stays the same.

## Use only `claude-or`, without alias commands

If you'd rather not have `ccds` and the others in your `PATH`, turn them off. Name the ones to
drop, or give no names to drop all of them (it asks first, `-y` skips the question):

```bash
claude-or --unlink ccfree ccglm    # just these
claude-or --unlink                 # all: "This removes the symlinks of all 6 aliases ... [y/N]"
```

It deletes their symlinks (only symlinks that point to `claude-or`) and marks the aliases
`"link": false` in `aliases.json`; the aliases themselves stay. You pass the alias or model to
`claude-or` instead:

```bash
claude-or ccds                     # alias from aliases.json
claude-or deepseek/deepseek-v4-pro # or the model id
```

Turned-off aliases stay unlinked through `--sync`, `--add`, `--remove` and `--self-update`.
To bring them back: `claude-or --link ccfree` for one, or `claude-or --link` for all (asks first).

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

## Windows and PowerShell

Looking for a similar tool for Windows and PowerShell? See
[win-claude-or](https://github.com/tatonkoduje/win-claude-or).
