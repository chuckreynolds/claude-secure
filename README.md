# Claude Code Safer Setup

Block Claude Code from reading `.env`, secrets, and credential files via permission rules in `~/.claude/settings.json`. Self-reference + something to share with teammates.

## TL;DR

Add a `permissions.deny` block to your global Claude Code settings. Claude will refuse to Read/Edit/Write or `cat`/`grep` matching files — no prompt, no override.

## Why

By default, granting Claude Code access to a project directory grants access to *everything* inside it, including `.env`. There's no built-in secret-file exclusion. One debugging session where Claude decides to "check the config" can land your secrets in a transcript that's synced to the cloud.

Three failure modes the deny rules prevent:

- **Direct read**: Claude opens `.env` to inspect a config var.
- **Shell escape**: Claude runs `cat .env` to bypass a Read deny.
- **Search leak**: Claude runs `rg DATABASE_URL .` and the matched file is `.env`, with the value printed in tool output.

## Install

1. Make sure `~/.claude/settings.json` exists:

   ```bash
   mkdir -p ~/.claude && [ -f ~/.claude/settings.json ] || echo '{}' > ~/.claude/settings.json
   ```

2. Open it and merge the `permissions.deny` block from [the snippet below](#the-settingsjson-snippet). **Don't replace the file** — preserve any existing keys (`enabledPlugins`, `attribution`, etc.). If you've already got a `permissions` block, append to its `deny` array.

3. (Optional) Create `~/.claude/rules/preferences.md` — see [second defense](#second-defense-preferencesmd) below.

4. Verify:

   ```bash
   jq '.permissions.deny | length' ~/.claude/settings.json
   ```

5. Smoke-test in a Claude Code session: ask "show me the contents of my .env file." It should refuse.

## What's blocked

| File pattern | Examples blocked | Why |
|---|---|---|
| `**/.env?(.!(example))` | `.env`, `.env.local`, `.env.production` | App secrets — explicitly **allows** `.env.example` |
| `**/*.tfvars` | `terraform.tfvars`, `prod.tfvars` | Cloud credentials in Terraform vars |
| `**/.tokens.json` | persisted JWT caches | Custom convention used in some projects |
| `**/credentials.json` | GCP / service-account JSON | Common credential filename |
| `**/*.pem` | TLS keys, SSH PEM keys | Private keys |
| `**/id_rsa*` | `id_rsa`, `id_rsa.pub` | Default SSH private key |

For each pattern, four classes of rule are added:

1. `Read(...)` — blocks the Read tool directly
2. `Edit(...)` — blocks Edit
3. `Write(...)` — blocks Write (also prevents *creating* a malicious one in your tree)
4. `Bash(cat|head|tail|less|more|grep|rg ...)` — closes the shell escape hatch

## The settings.json snippet

```json
{
  "permissions": {
    "deny": [
      "Read(**/.env?(.!(example)))",
      "Edit(**/.env?(.!(example)))",
      "Write(**/.env?(.!(example)))",
      "Bash(cat **/.env?(.!(example)))",
      "Bash(head **/.env?(.!(example)))",
      "Bash(tail **/.env?(.!(example)))",
      "Bash(less **/.env?(.!(example)))",
      "Bash(more **/.env?(.!(example)))",
      "Bash(grep **/.env?(.!(example)))",
      "Bash(rg **/.env?(.!(example)))",

      "Read(**/*.tfvars)",
      "Edit(**/*.tfvars)",
      "Write(**/*.tfvars)",
      "Bash(cat **/*.tfvars)",
      "Bash(head **/*.tfvars)",
      "Bash(tail **/*.tfvars)",
      "Bash(less **/*.tfvars)",
      "Bash(more **/*.tfvars)",
      "Bash(grep **/*.tfvars)",
      "Bash(rg **/*.tfvars)",

      "Read(**/.tokens.json)",
      "Edit(**/.tokens.json)",
      "Write(**/.tokens.json)",
      "Bash(cat **/.tokens.json)",
      "Bash(head **/.tokens.json)",
      "Bash(tail **/.tokens.json)",
      "Bash(less **/.tokens.json)",
      "Bash(more **/.tokens.json)",

      "Read(**/credentials.json)",
      "Edit(**/credentials.json)",
      "Write(**/credentials.json)",
      "Bash(cat **/credentials.json)",

      "Read(**/*.pem)",
      "Edit(**/*.pem)",
      "Write(**/*.pem)",
      "Bash(cat **/*.pem)",

      "Read(**/id_rsa*)",
      "Edit(**/id_rsa*)",
      "Write(**/id_rsa*)",
      "Bash(cat **/id_rsa*)"
    ]
  }
}
```

## The `.env.example` exception

Pattern: `**/.env?(.!(example))`

Reading it left-to-right:

- `**/` — at any directory depth
- `.env` — must start with literal `.env`
- `?(...)` — optionally followed by ONE more chunk (picomatch extglob: zero-or-one)
- `.!(example)` — that chunk is `.` plus *anything except exactly* `example`

Test cases (verified against picomatch):

| File | Result |
|---|---|
| `.env` | blocked |
| `.env.local` | blocked |
| `.env.production` | blocked |
| `.env.production.local` | blocked |
| `foo/bar/.env` | blocked (any depth) |
| **`.env.example`** | **allowed** |
| **`foo/.env.example`** | **allowed** |
| `.env.example.bak` | blocked (backups of secret files stay protected) |

Why allow `.env.example`? It's intended to be committed and contains *placeholder* values — letting Claude read it gives useful context (which env vars a service expects, default values, format hints) without exposing real secrets.

If you'd rather lock it down completely with no exception, replace the pattern with `**/.env*` (drop the `?(.!(example))` suffix from all 10 rules).

## Second defense: `preferences.md`

The deny rules are the hard gate — they block Claude even when it doesn't know the file is sensitive. A preferences file nudges Claude toward the same outcome *before* it tries, often with a more graceful "I'll skip this" rather than a hard permission error.

Create `~/.claude/rules/preferences.md`:

```markdown
# NEVER

- Read `.env` files (`.env.example` OK)
- Read `.tfvars` files
- Read files that may contain secrets — ask first
```

## Customize

**Add patterns** for your project's secret-file conventions. Common additions:

- `**/secrets.yaml`, `**/secrets.yml` — Helm / k8s
- `**/*.kdbx` — KeePass
- `**/*.priv` — generic private key
- `**/.netrc` — HTTP auth
- `**/.aws/credentials`, `**/.gcp/credentials` — cloud SDK creds
- `**/*service-account*.json` — GCP service account keys

For each: add `Read(...)`, `Edit(...)`, `Write(...)`, and at minimum `Bash(cat ...)`.

**Scope.** Everything above goes in `~/.claude/settings.json` (user-global, applies to every project). For per-project rules:

| File | Scope | Git |
|---|---|---|
| `~/.claude/settings.json` | All your projects | n/a |
| `<project>/.claude/settings.json` | That repo, for everyone | committed |
| `<project>/.claude/settings.local.json` | That repo, just you | gitignored |

Deny rules merge across all three sources.

## How permission rules work

| Field | Behavior |
|---|---|
| `allow` | Pre-approves the operation; no prompt |
| `ask` | Always prompts (overrides default approval) |
| `deny` | Refuses unconditionally — no prompt, no override |

Deny wins over allow. Patterns inside `(...)` use [picomatch](https://github.com/micromatch/picomatch) glob syntax with extglob enabled (`?(...)`, `!(...)`, `+(...)`, `@(...)`, `*(...)`).

## Contributing

Found a missed file pattern, a better approach, or want to add support for another tool? [Open an issue](https://github.com/chuckreynolds/claude-secure/issues).


---

_The `preferences.md` idea is from [Tips for safer Claude coding](https://anatolinicolae.com/tips-for-safer-claude-coding/) by Anatoli Nicolae._
