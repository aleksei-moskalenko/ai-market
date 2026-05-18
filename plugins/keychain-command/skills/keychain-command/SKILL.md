---
name: keychain-command
description: |
  Use when the user wants to (a) add a new template command that uses a
  secret stored in macOS Keychain so Claude executes it without ever
  seeing the secret, or (b) run / list / remove an already-registered
  secret command. Triggers in Russian/English: "добавь команду которая
  ...", "сохрани секрет для ...", "какие у нас secret-команды?", "secret
  command", "keychain wrapper", or whenever Claude is about to run a
  command (sql client, http call, cli tool) whose only credential should
  live in macOS Keychain. Use BEFORE suggesting any plaintext secret in
  chat or in a file.
argument-hint: add | run <name> [args...] | list | remove <name>
allowed-tools: Bash(mkdir -p ~/.claude-work/secret-commands/bin), Bash(touch ~/.claude-work/secret-commands/REGISTRY.md), Bash(ls ~/.claude-work/secret-commands/bin*), Bash(~/.claude-work/secret-commands/bin/*), Bash(security find-generic-password -a * -s claude.*), Bash(chmod +x ~/.claude-work/secret-commands/bin/*), Read, Edit, Write
---

# Keychain Command

A skill that **manages template commands which load their secrets from
macOS Keychain at exec time**. Claude calls the wrapper; the secret is
fetched inside the wrapper process and never enters the conversation
context.

## Why

- Claude never sees the plaintext secret in its tool I/O.
- macOS Keychain enforces an authorization prompt on every access (we use
  ACL `-T ""` — no trusted applications, so each call prompts).
- Wrappers are versioned shell scripts in a known location, easy to
  audit and parameterise.

## Strict role split

| Role   | Responsibility                                                                                |
| ------ | --------------------------------------------------------------------------------------------- |
| Claude | Generates wrappers, edits `REGISTRY.md`, runs wrappers, reads Keychain **existence** only.    |
| User   | Adds, updates, and deletes Keychain entries themselves — in their own terminal, not via chat. |

Claude MUST NOT:
- Run `security add-generic-password`, `security delete-generic-password`,
  or any `security` subcommand that writes / reads the secret value.
- Run `osascript` to prompt for a password.
- Accept the secret pasted into chat. If the user pastes a secret,
  refuse to use it, ask them to put it in Keychain themselves, and
  recommend they rotate the value they just leaked.
- Echo, log, `cat`, or otherwise reveal anything that could be a secret.

Claude MAY:
- Run `security find-generic-password -a "$USER" -s "claude.<name>"` with
  no `-w` / `-g` flags to **check existence** of an entry (no secret
  returned).
- Run the wrappers themselves (which read the secret inside the wrapper
  subprocess and use it as an env var).

## Layout

```
~/.claude-work/secret-commands/
  REGISTRY.md         # human/Claude-readable index of wrappers
  bin/                # generated wrappers, chmod +x
    <name>            # e.g. sql-home-pg, gh-personal
```

Keychain entries use service name `claude.<name>` (`security` `-s` flag),
account = `$USER`.

This storage is **shared** with any other keychain-wrapper skills on the
machine (e.g. team-specific `gm-keychain-command`). Wrappers and
`REGISTRY.md` entries created by one skill are visible to the others.
Pick wrapper names accordingly (`sql-personal-pi`, `gh-personal`, etc.)
to avoid collisions with work entries.

## First-time setup

If `~/.claude-work/secret-commands/bin/` does not exist, create the layout:

```bash
mkdir -p ~/.claude-work/secret-commands/bin
touch ~/.claude-work/secret-commands/REGISTRY.md
```

That is the entire setup. Wrappers are always invoked by **full path**:
`~/.claude-work/secret-commands/bin/<name>`. Do NOT modify the user's
shell profile (`.zshrc`, `.bashrc`, etc.) — the absolute path is
deterministic, so PATH manipulation is unnecessary and adds risk.

## Subcommand: `add` — register a new secret command

Two phases. Claude does the script work; the user touches Keychain
themselves.

### Phase A — Claude

1. Collect from the user, in one block, the following:
   - `name` — slug matching `^[a-z0-9-]+$`, e.g. `sql-home-pg`.
   - `description` — one line, what this wrapper does.
   - `template` — one of: `psql`, `mysql`, `redis-cli`, `http-header`,
     `env-var`. See Templates section.
   - Template-specific params (host, port, user, database, env name, …).
   - `keychain_service` — defaults to `claude.<name>`; only override
     when reusing an existing entry.
   - **Do NOT ask for the secret. Refuse if the user offers it.**

2. Check existence (no secret returned):
   ```bash
   security find-generic-password -a "$USER" -s "claude.<name>" >/dev/null 2>&1 && echo EXISTS || echo NEW
   ```
   If `EXISTS`, ask the user whether they want to keep / overwrite —
   only affects which `security` command Claude shows them in Phase B.

3. Generate the wrapper from the chosen template (see Templates), write
   to `~/.claude-work/secret-commands/bin/<name>`, then `chmod +x`.

4. Append to `REGISTRY.md`:
   ```markdown
   ## <name>
   - Description: <description>
   - Template: <template>
   - Keychain service: claude.<name>
   - Wrapper: ~/.claude-work/secret-commands/bin/<name>
   - Created: <YYYY-MM-DD>
   - Usage: <one-line example>
   ```

5. Print the **exact command** the user needs to run in their own
   terminal (Terminal.app or iTerm — **NOT** via the `!` prefix in this
   chat, since that echoes through Claude's tool I/O):

   New entry:
   ```bash
   read -rs "?Secret for claude.<name>: " PW && echo \
     && security add-generic-password -a "$USER" -s "claude.<name>" -T "" -w "$PW" \
     && unset PW && echo "Stored."
   ```

   Overwrite existing:
   ```bash
   read -rs "?Secret for claude.<name>: " PW && echo \
     && security add-generic-password -U -a "$USER" -s "claude.<name>" -T "" -w "$PW" \
     && unset PW && echo "Updated."
   ```

   Note: `read -rs "?...":` is zsh syntax (this machine uses zsh). For
   bash use `read -rsp "Secret: " PW`.

### Phase B — User

User opens their own terminal, pastes the command from step 5, types
the secret (input is hidden), presses Enter. Done. Claude has zero
involvement.

### Phase C — Claude verifies

After the user confirms they ran the command:

1. Re-check existence (still no secret returned):
   ```bash
   security find-generic-password -a "$USER" -s "claude.<name>" >/dev/null 2>&1 && echo OK || echo MISSING
   ```
2. Smoke-test the wrapper with a harmless call (e.g. `SELECT 1` for
   `psql`, `PING` for `redis-cli`). On first call, macOS shows a
   Keychain authorisation prompt — that is expected and proves ACL is
   correct.

## Subcommand: `run <name> [args...]`

Call the wrapper by its full absolute path:
`~/.claude-work/secret-commands/bin/<name>`. Never inline the secret.

If `<name>` is unknown, read `REGISTRY.md` and tell the user what is
available.

## Subcommand: `list`

Read `REGISTRY.md` and present a compact table: name, description,
template, keychain service. If `REGISTRY.md` is missing or empty, say so.

## Subcommand: `remove <name>`

Same two-phase split.

### Phase A — Claude

1. Confirm with the user.
2. `rm ~/.claude-work/secret-commands/bin/<name>`.
3. Edit `REGISTRY.md` to drop the section.
4. Print the command for the user to run themselves to delete the
   Keychain entry:
   ```bash
   security delete-generic-password -a "$USER" -s "claude.<name>"
   ```

### Phase B — User

Runs the command in their own terminal.

### Phase C — Claude verifies

```bash
security find-generic-password -a "$USER" -s "claude.<name>" >/dev/null 2>&1 && echo STILL_EXISTS || echo GONE
```

## Templates

All wrappers share this preamble:

```bash
#!/bin/bash
# claude-keychain-command: <name>
set -euo pipefail
KC_SERVICE="claude.<name>"
get_secret() { security find-generic-password -a "$USER" -s "$KC_SERVICE" -w; }
```

### `psql`

Params: `host`, `port` (default `5432`), `user`, `database`,
optional `sslmode`.

```bash
PG_HOST=<host>; PG_PORT=<port>; PG_USER=<user>; PG_DB=<database>
[[ "${1:-}" == "" ]] && { echo "Usage: <name> '<SQL>' [extra psql args]" >&2; exit 2; }
SQL="$1"; shift
PGPASSWORD="$(get_secret)" \
  psql -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" -d "$PG_DB" \
       ${SSLMODE:+--set=sslmode="$SSLMODE"} \
       -v ON_ERROR_STOP=1 "$@" -c "$SQL"
```

### `mysql`

```bash
MY_HOST=<host>; MY_PORT=<port>; MY_USER=<user>; MY_DB=<database>
[[ "${1:-}" == "" ]] && { echo "Usage: <name> '<SQL>' [extra mysql args]" >&2; exit 2; }
SQL="$1"; shift
MYSQL_PWD="$(get_secret)" \
  mysql -h "$MY_HOST" -P "$MY_PORT" -u "$MY_USER" "$MY_DB" "$@" -e "$SQL"
```

### `redis-cli`

```bash
REDIS_HOST=<host>; REDIS_PORT=<port>
get_secret | redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" -a - --no-auth-warning "$@"
```

### `http-header`

For tools that authenticate with a single header.
Params: `header_name` (e.g. `Authorization`), `prefix` (e.g. `Bearer `).

```bash
HEADER_NAME='<header_name>'; PREFIX='<prefix>'
curl -sS -H "$HEADER_NAME: $PREFIX$(get_secret)" "$@"
```

### `env-var`

For any command — secret exposed as a named env var to the child only.
Params: `env_name` (e.g. `API_TOKEN`), `command` (e.g. `gh`, `stripe`).

```bash
export <env_name>="$(get_secret)"
exec <command> "$@"
```

## Security notes

- ACL `-T ""` means **no trusted applications** — Keychain prompts for
  user auth on every access. That is the desired UX. Do NOT add a trusted
  app with `-T <path>` to silence the prompt.
- Postgres/MySQL receive the password via env var (`PGPASSWORD`,
  `MYSQL_PWD`), NOT argv — `ps` will not show it.
- `redis-cli -a -` reads the password from stdin (we pipe `get_secret`)
  to avoid argv exposure.
- Never write the secret to disk, even temporarily.
- Wrapper stdout/stderr IS captured by Claude — never `echo "$PW"` for
  debugging.
- On corporate Macs with FileVault + Touch ID, the Keychain prompt may
  accept Touch ID where the OS allows.

## When NOT to use

- Long-running daemons started before the user is present — the Keychain
  prompt will block forever.
- Secrets that must be shared across machines — Keychain is local.
- If 1Password CLI (`op run`) is already set up — usually more ergonomic;
  prefer that and skip this skill.

## Common mistakes

| Mistake                                              | Why it fails                              | Fix                                |
| ---------------------------------------------------- | ----------------------------------------- | ---------------------------------- |
| `echo "$PW"` while debugging                         | Secret leaks into the conversation        | Never log the secret variable      |
| `-T /usr/bin/psql` on `security add-generic-password` | No auth prompt — defeats the purpose      | Use `-T ""`                        |
| `psql -W ...` instead of `PGPASSWORD=...`            | `-W` prompts on TTY; wrapper has none     | Pass via `PGPASSWORD` env var      |
| One wrapper handles dev+prod via a flag              | Wrong env on autopilot → wrong DB hit     | One wrapper per env (`sql-home`, `sql-side-project`) |
| Storing the secret inside the wrapper script         | Defeats the entire skill                  | Always fetch via `get_secret`      |

## Red flags — STOP

- The user pastes a real secret into chat → refuse to use it. Tell them
  to put it in Keychain themselves and rotate the leaked value.
- You're about to run `security add-generic-password` / `security
  delete-generic-password` / `osascript display dialog ... hidden
  answer` → STOP. That's the user's job. Print the command for them.
- A wrapper file contains a hex/base64 blob in its source → it's been
  tampered with; regenerate from the template.
- `security` calls do NOT trigger a UI prompt → ACL is wrong; tell the
  user to recreate the Keychain entry with `-T ""`.
