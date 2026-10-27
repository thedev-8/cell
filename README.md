# cell — isolated environments for AI agent CLIs

Like Python venvs, but for AI runtimes. Each agent gets its own config,
API key, model, memory and persona — scoped to the project you run it in.
Nothing leaks between agents, and nothing touches your global setup.

A self-hosted, fully-auditable alternative to [helo](https://github.com/ryha0008-boop/helo-win).

## Why this exists (and why it's different from helo)

| | helo | cell |
|---|---|---|
| Code | Compiled Rust binary | **One Python file, stdlib only** — read it all, edit it, no build step |
| Adding a runtime | Edit source in ~12 places, rebuild | `cell runtime add` — **runtimes are data, not code** |
| Adding an LLM subscription | Hardcoded provider list | `cell provider add` — **any Anthropic-compatible endpoint**, 10 presets built in |
| Model | Stored, never applied | Applied: seeded into `settings.json` (claude) or passed as a flag (kimi, custom) |
| Bad runtime/provider name | Accepted silently, fails at `run` | **Fails at `add` time** with the list of valid names |
| Before launch | Nothing | `cell show` — **dry-run: files, env, command, secrets masked** |
| Secrets | Written into the project's `settings.json` | **Only in the process env at launch**; never inside the project |
| Git | Env dirs get committed (they hold your logins) | Added to `.git/info/exclude` automatically, from any subfolder or worktree; `doctor` flags leaks |
| Health check | — | `cell doctor` |
| Voice input | — | `cell voice`: speak prompts, transcribe voice notes, local Whisper or any speech API |
| Migrate from helo | — | `cell import` reads your helo config and `.helo.toml` instances |

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/thedev-8/cell/main/cell -o ~/bin/cell   # or ~/.local/bin — anywhere in PATH
chmod +x ~/bin/cell
cell init
```

Or clone this repo and copy the single `cell` file. Update later by re-running the same `curl` line.

Requires Python 3.11+ (for `tomllib`). No other dependencies, ever.

## Quick start

```bash
# Claude Max subscription (no key: Claude Code signs you in once per env)
cell add max --runtime claude --provider anthropic --model opus --login

# Z.ai GLM subscription, same Claude Code runtime
cell add glm --runtime claude --provider zai --model glm-5.3
cell keys set zai                       # prompts for the key, hidden

# Kimi Code subscription inside the kimi runtime
cell add kimicode --runtime kimi --provider kimi --model kimi-code/k3 --login

cd my-project
cell show glm                           # dry-run: what will happen, secrets masked
cell run glm                            # creates ./.claude-env-glm/, launches Claude Code on GLM
cell max                                # short for: cell run max
cell max -p "explain this codebase"
cell max -c                             # continue the last session
cell max -r                             # pick a session to resume
cell sessions                           # every session in this folder, with titles; type a number to open
cell                                    # interactive menu
```

Missing flags have sensible defaults: `cell add glm --provider zai` picks the
claude runtime and the provider's default model, and says so. In a terminal,
`cell add NAME` asks for each field and re-asks on a typo.

## Sessions

```bash
cell sessions                 # sessions started in this folder, newest first
cell sessions max -n 30       # one blueprint, more history
cell sessions --all           # include sessions started in other folders
cell max -r 3f2a9c1e          # open one: the 8-character id from the list is enough
cell max -r                   # or the runtime's own picker
cell max -c                   # or simply the latest
```

```
max (claude)
    1  2026-10-27 03:44  3f2a9c1e  Fix login redirect loop
    2  2026-10-24 09:16  b71d04aa  parser-refactor
kimicode (kimi)
    3  2026-09-23 23:10  5c8e2d90  add tests for the csv importer

open one:  cell max -r 3f2a9c1e      latest:  cell max -c
open # (Enter = quit):
```

Titles come from the runtime itself: your own `/rename` title first, then
Claude Code's generated title (kimi: its session title), then the last prompt.
In a terminal, type a number and that session opens in the right runtime
with the right env dir. A short id works everywhere `-r` does, and
`cell run -r ID` without a name finds the blueprint that owns the session.

## Any LLM subscription

A *provider* is an LLM subscription or API account. Built in, with the
endpoints taken from each vendor's own Claude Code guide:

| provider | works with claude runtime via | key env |
|---|---|---|
| `anthropic` | first-party (login or `ANTHROPIC_API_KEY`) | `ANTHROPIC_API_KEY` |
| `zai` | `https://api.z.ai/api/anthropic` | `ZAI_API_KEY` |
| `moonshot` | `https://api.moonshot.ai/anthropic` | `MOONSHOT_API_KEY` |
| `deepseek` | `https://api.deepseek.com/anthropic` | `DEEPSEEK_API_KEY` |
| `minimax` | `https://api.minimax.io/anthropic` | `MINIMAX_API_KEY` |
| `openrouter` | `https://openrouter.ai/api` | `OPENROUTER_API_KEY` |
| `qwen` | `https://coding-intl.dashscope.aliyuncs.com/apps/anthropic` | `DASHSCOPE_API_KEY` |
| `ollama` | `http://localhost:11434` (no key) | — |
| `kimi` | — (login inside the `kimi` runtime) | — |
| `openai` | — (no Anthropic-compatible endpoint; use a proxy or the `kimi` runtime) | `OPENAI_API_KEY` |

Anything else is one command:

```bash
cell provider add myproxy --base-url http://localhost:4000 --key-env MYPROXY_KEY \
    --tier haiku=small-model --env API_TIMEOUT_MS=600000
cell add fast --runtime claude --provider myproxy --model big-model
```

`cell provider list` shows every provider, its endpoint, whether a key is
stored and which blueprints use it. `cell provider show zai` prints the
exact table; `cell provider add zai --force --tier haiku=glm-5.3-flash`
overrides a single field of a built-in.

What the claude runtime gets for a third-party provider:

- `settings.json` → `env`: `ANTHROPIC_BASE_URL`, `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU,FABLE}_MODEL`
  (your model, or the provider's `tiers`), plus any provider `env` extras.
  Non-secret, visible, auditable.
- process env at launch: `ANTHROPIC_AUTH_TOKEN` (or `ANTHROPIC_API_KEY` for
  `--auth api-key` providers). Never written inside the project.
- any ambient `ANTHROPIC_API_KEY` / `ANTHROPIC_BASE_URL` in your shell is
  removed from the child so it can't misroute or mis-bill the session.

For the kimi runtime, providers with a `kimi_catalog` id are registered with
`kimi provider catalog add` the first time you run the blueprint.

## Commands

```
cell                        interactive menu: run, resume, continue, sessions, show, add, edit, delete
cell init                   first-time setup
cell add <n> [--runtime R] [--provider P] [--model M] [--login | --api-key K | --api-key-env VAR]
                            [--persona T|FILE] [--permissions bypass|accept-edits|plan|auto|default]
                            (missing flags are asked for in a terminal, or defaulted)
cell edit <n> [same flags; '' clears] [--no-hooks [EVENT...]] [--hooks [EVENT...]]
                            (no flags: shows the blueprint and what you can change)
cell list | ls
cell remove | rm <n>
cell <n> [run options]      short for: cell run <n>
cell run [n] [-r [ID] | -c] [-p prompt] [--voice] [--model M] [--api-key K] [-- extra args]
                            no name: the blueprint with an env dir here, else asks
cell show [n] [...]         dry-run of `run`: files, env, command, secrets masked
cell status                 config path, runtimes, instances here
cell doctor                 health check: config, runtimes, keys, instances, git
cell keys [set|list|remove]          global keys, one per provider (set prompts if no key given)
cell key <n> [key]                   per-blueprint key
cell provider [list|add|show|remove]
cell runtime [list|add|install|remove]
cell hooks [list|set|remove|import]
cell templates [list|show]           personas: coding, assistant, devops + ~/.config/cell/templates/*.md
cell voice [setup|listen|transcribe FILE…]   speak prompts, transcribe voice notes (local or API)
cell sessions [n] [--all] [-n N]     sessions in this folder with titles; type a number to open
cell plugins [add|remove NAME…]     Claude Code plugins every claude env installs on its next run
cell learn [on|off|now|add RULE|remove N|log]  self-improving instructions from your corrections
cell clean [n] [-y] [--global RUNTIME]
cell import [dir] [--config helo.toml] [--recursive] [--force] [--dry-run]
```

`keys`, `provider`, `runtime`, `hooks` and `templates` list by default;
`cell voice` alone shows the voice setup. A mistyped name gets a "did you
mean" with the valid choices.

## How isolation works

Each runtime knows one environment variable that relocates its home directory:

- `claude` → `CLAUDE_CONFIG_DIR`
- `kimi` → `KIMI_CODE_HOME`

`cell run max` creates `.claude-env-max/` in the current project, merges
cell-managed keys into `settings.json` (`env`, `model`, `permissions`,
`hooks` — everything else in the file is left alone), writes or links the
persona file, then **replaces itself** with the runtime (`exec`), so Ctrl-C
goes to the runtime alone. That's the whole trick — verify it with
`cell show max`.

Model handling is deliberately gentle: the blueprint model is seeded into a
new env, and re-applied only when you `cell edit --model`. Your in-app
`/model` choice survives between runs. Runtimes without a settings file
(kimi, custom) get the model as a flag on every launch; `cell run --model X`
overrides for one session.

Add a runtime that isn't built in:

```bash
cell runtime add opencode --binary opencode --env-var OPENCODE_CONFIG_DIR \
    --model-flag=--model --prompt-flag=run
cell add oc --runtime opencode --provider openai --model gpt-5.5
```

A custom runtime receives the provider's key as its `key_env` variable
(`OPENAI_API_KEY` here) plus the relocation variable; nothing else is assumed.

## Hooks and seed files

Claude Code hooks you want in *every* env live once in your config:

```bash
cell hooks set stop 'git add -A && git commit -qm "auto: $(date -u +%FT%TZ)" || true'
cell hooks import .claude-env-max     # or copy them from an env helo made
cell edit max --no-hooks stop          # per-blueprint opt-out
```

`cell hooks import` also copies the env's `hooks/` scripts into
`~/.config/cell/seed/claude/`. Everything under `seed/<runtime>/` is copied
into each new env dir of that runtime (never overwriting), so hook scripts,
a `settings.json` baseline, or anything a runtime expects is there on first
launch.

## Self-improving instructions

```bash
cell learn on        # turn it on (needs a claude blueprint with a persona file)
cell learn           # rules learned so far, each with the words of yours that prompted it
cell learn remove 2  # drop one
cell learn add "Never push without asking."   # add your own
cell learn now       # analyse the latest session in this folder right away
cell learn log       # what happened after each session
cell learn off
```

When a Claude Code session ends, a `SessionEnd` hook hands it to cell in the
background, so quitting stays instant. cell asks the same env, headless with
no tools, no hooks and nothing saved, which corrections or lasting
preferences you expressed. At most three short rules come back per session.
Each must quote your own words, so nothing is inferred from what the
assistant did by itself. Rules are written into a marked block at the end of
the blueprint's persona `CLAUDE.md`, so every later session follows them:

```
<!-- cell:learned:start … -->
## Learned from past sessions

- Show the exact working command in every error message.
<!-- cell:learned:end -->
```

Sessions with fewer than three prompts are skipped, a session is analysed
only once (a resumed one only for its new prompts), and the list stops at
25 rules. Each analysis is one short `sonnet` call on that env's login.

## Voice input

Speak a prompt instead of typing it, or hand the agent a voice note. Whisper
runs locally by default (private, offline, no key); any OpenAI-compatible
speech API works too.

```bash
cell voice setup                          # own venv + faster-whisper, model "small", /voice skill
cell voice listen                         # records until you pause 1.5 s (or Enter), prints the text
cell voice transcribe note.ogg memo.m4a   # voice notes from your phone -> text
cell run max --voice                      # speak the opening prompt, then the session starts
cell run max -p "$(cell voice listen -q)" # one-shot: status goes to stderr, text to stdout

cell voice setup --model turbo            # bigger/better models: base, small, medium, large-v3, turbo
cell voice setup --language hi            # force a language (default: auto-detect)
cell voice setup --engine api --api-url https://api.groq.com/openai/v1 \
    --api-model whisper-large-v3-turbo --api-key-env GROQ_API_KEY     # or: cell keys set voice
```

Inside Claude Code every env gets a `/voice` skill (seeded from
`~/.config/cell/seed/claude/skills/voice/`): type `/voice` and speak, or paste
a voice-note path and say "listen to this"; the agent runs the commands above
and acts on the transcript. For other runtimes, add the same two commands to
the persona file.

Recording uses `parecord`, `arecord` or `pw-record` (whichever exists), 16 kHz
mono. A custom recorder is one config line:
`recorder = "ffmpeg -f alsa -i hw:1 -ac 1 -ar 16000 -f s16le -"`. `cell doctor`
checks the engine, model and recorder; `listen` refuses a recording that is
pure silence (typical for a VM without audio input enabled).

## Your config

`~/.config/cell/config.toml` (override with `$CELL_HOME`). Plain TOML, hand-editable, chmod 600:

```toml
[blueprints.max]
runtime = "claude"
provider = "anthropic"
model = "opus"
auth = "login"               # subscription login; no key anywhere
persona = "/home/me/.config/cell/templates/coding.md"
permissions = "bypass"

[blueprints.glm]
runtime = "claude"
provider = "zai"
model = "glm-5.3"

[keys]
zai = "..."

[providers.myproxy]          # any Anthropic-compatible endpoint
base_url = "http://localhost:4000"
key_env = "MYPROXY_KEY"
auth = "token"

[runtimes.opencode]          # any CLI
binary = "opencode"
env_var = "OPENCODE_CONFIG_DIR"
model_flag = "--model"

[hooks]
stop = "..."

[voice]                      # written by `cell voice setup`
engine = "local"             # or "api"
model = "small"
language = ""                # auto-detect
```

## API key priority

`--api-key` flag > blueprint literal > blueprint `--api-key-env` var > global `keys` >
provider env var (`ANTHROPIC_API_KEY`, `ZAI_API_KEY`, …) > provider fixed token (ollama).

`--login` blueprints skip all of that: the runtime signs in itself and any
ambient key is stripped from its environment. Prefer `--api-key-env MY_KEY`
if you don't want keys on disk at all.

## Migrating from helo

```bash
cell import --dry-run      # reads ~/.config/helo/config.toml and ./*/.helo.toml
cell import                # blueprints, personas (claude_md), permissions, keys
cell hooks import .claude-env-max
cell doctor
```

helo's `api_key = "max"` becomes `auth = "login"` — whether it sits in the
blueprint or once per provider under helo's `[keys]`. A login-capable provider
(anthropic, kimi) with no key anywhere is imported as login too. Existing env dirs are
reused as they are; cell only touches the keys it manages.

## License

MIT
