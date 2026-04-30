---
name: manage-credentials
description: Create or update the encrypted credentials file (credentials.env.age) in the client's harness repo. Use when setting up API keys for the first time or adding/rotating a key. Run by the client admin.
---

# manage-credentials

**Execute this skill directly. Do not enter plan mode. Do not expose internal derivation steps to the user.** The user should see only: what you found, what you need from them, and the finished result.

## When to use

- First-time credential setup for a client (no `credentials.env.age` yet in their harness repo).
- Adding a new API key (e.g., a new integration).
- Rotating an existing key.

## When NOT to use

- You are a client employee. This skill manages the credentials file — employees don't do that, their admin does.
- `age` is not installed on your machine. Install it first: `brew install age` (Mac) or `winget install FiloSottile.age` (Windows).

---

## Canonical operator

**This skill is run by the client admin on their own machine.** The consultant's role ends at `/onboard-client`; credential population happens admin-side so the admin controls their own API keys and their own passphrase.

The consultant may run this skill from the master repo for demos, drills, or emergency rotations — that's not the canonical path, and the pre-flight gate below permits both.

---

## Credential channel (admin's choice)

The admin chooses how to distribute the passphrase + installer to employees. Options in order of security:

- **1Password / Bitwarden shared items** — encrypted at rest, access-controlled per person. Preferred.
- **Encrypted email** (S/MIME, PGP, password-protected archive).
- **Direct messaging in a secured workspace** (Slack DM in a company workspace, Teams private chat).

The skill does not prescribe a channel. Use whatever matches your org's existing secrets-distribution conventions. Do not paste the passphrase in the same message as the installer one-liner unless the channel is end-to-end encrypted.

---

## Pre-flight gate (run silently before anything else)

This skill manages secrets and must only run on admin/consultant machines. Matches the `/client-admin:generate-installer` gate: `clients.json` is the only automatic signal that survives; everything else falls back to a single conversational question.

```bash
# Signals:
#   - clients.json in $PWD                       → consultant (certain)
#   - .claude-plugin/marketplace.json in $PWD    → admin OR consultant
#     (admin working inside their harness checkout, or consultant inside a client repo)
#   - Neither present                             → unknown — ask the user
if [ -f clients.json ]; then
  ROLE=consultant
elif [ -f .claude-plugin/marketplace.json ]; then
  ROLE=admin_or_consultant
else
  ROLE=unknown
fi
```

If `ROLE=unknown`, ask the user ONE question (no multi-step triage):

> Quick check before I manage credentials — who's running this: **admin**, **consultant**, or **employee**?

- "admin" or "consultant" → proceed to client detection below.
- "employee" (or any equivalent — "team member", etc.) → refuse with the message below and stop.
- "not sure" → ask them to clarify; default to refuse if still unclear.

Employee refusal text:

> This skill manages the encrypted credentials file for the client harness — it's for the admin or consultant, not team members. Contact your admin if you need a key added or rotated.

---

## Resolve target client repo

**Context A — Master repo (consultant):** `clients.json` exists in `$PWD`. List active clients by human name — titlecase the `name` field (e.g., `test-client` → "Test Client"). Ask which client if more than one.

**Context B — Client repo (admin):** `.claude-plugin/marketplace.json` exists in `$PWD`. Extract `<org>/<repo>` from the git remote URL.

**Context C — Installed harness (admin's own machine):** Run this exact bash to build the candidate list — only client harness repos have a `credentials/` directory; marketplace catalogs do not:

```bash
for name in $(jq -r 'keys[]' ~/.claude/plugins/known_marketplaces.json 2>/dev/null); do
  [ -d "$HOME/.claude/plugins/marketplaces/$name/credentials" ] && echo "$name"
done
```

Each line of output is a valid client harness name. If no lines are output, fall through to the prompt below. For each candidate, read the human-friendly name from `~/.claude/plugins/marketplaces/<name>/.claude-plugin/marketplace.json` → `.name` field, titlecased. If exactly one candidate, use it automatically. If multiple, ask which.

If none of A/B/C produces a repo, ask: "I couldn't detect a client repo. Are you running this from your master repo, from inside a client's harness repo, or on a machine with the harness installed?"

---

## Passphrase security rule (non-negotiable)

Passphrase input is handled by `age` natively — it opens `/dev/tty` directly and does not echo. **Never ask for a passphrase as a chat message. Never include any passphrase value in tool output, responses, chain-of-thought, or subagent prompts.** The passphrase never enters Claude's context.

---

## Workflow

### Step 1: Clone client repo

Admin machines have `gh` authenticated — use `gh repo clone` so no token is embedded in a URL or echoed.

```bash
TMP_WORK=$(mktemp -d)
trap 'rm -rf "$TMP_WORK"' EXIT
gh repo clone "${CLIENT_REPO}" "$TMP_WORK/repo" -- --depth 1
```

If `gh repo clone` fails (e.g., `gh` not installed), fall back to:

```bash
git clone --depth 1 "https://github.com/${CLIENT_REPO}.git" "$TMP_WORK/repo"
```

(The credential helper configured by bootstrap will handle auth. Do NOT construct a token-in-URL string or echo any token value.)

### Step 2: Load existing keys (if any)

If `$TMP_WORK/repo/credentials/credentials.env.age` exists, ask the user in chat:

> I found an existing credentials file. Do you know the current passphrase?
> 1. Yes — enter it now to load and review the current keys before editing
> 2. No / Start fresh — replace the file with new credentials (existing keys will be lost)

If **Yes**: execute via Bash tool — `age` will prompt for the passphrase at the terminal (no echo):

```bash
age --decrypt -o "$TMP_WORK/credentials.env" "$TMP_WORK/repo/credentials/credentials.env.age"
```

Display the current keys, **masking values**:

```
Current keys:
  TAVILY_API_KEY = ****
  AVOMA_API_KEY  = ****
```

If **No / Start fresh**: skip decryption, start with an empty credentials set. Tell the user: "Starting fresh — existing credentials will be replaced when you save."

If no existing file: start with an empty credentials set silently.

### Step 3: Collect keys via terminal commands

Ask the user which API keys they want to add or update. For each key they name, output a copy-paste terminal command in triple backticks so the value is entered securely (not visible, not in chat, not in shell history). Use `printf` for the prompt and `read -rs` without `-p` — this works in both bash and zsh (`-p` in zsh means "read from coprocess", not "print prompt"):

```bash
printf "TAVILY_API_KEY: " && read -rs v && echo && echo "TAVILY_API_KEY=$v" >> "$TMP_WORK/credentials.env"
```

Replace `TAVILY_API_KEY` with the actual key name. Provide one command block per key. After the admin pastes and runs each command, ask "Any other keys? (say done when finished)".

To remove a key (start-fresh path or explicit removal), omit it — don't append it to credentials.env.

The final `$TMP_WORK/credentials.env` must never be printed or logged.

### Step 4: Encrypt with admin-chosen passphrase

Run via Bash tool. `age` will prompt for the passphrase and confirmation at the terminal — no echo, never enters Claude's context. The admin picks the passphrase from their password manager and types it when prompted.

```bash
mkdir -p "$TMP_WORK/repo/credentials"
age --encrypt --passphrase -o "$TMP_WORK/repo/credentials/credentials.env.age" "$TMP_WORK/credentials.env"
```

### Step 4b: Drop plaintext at canonical path + wire admin's shell rc

The admin already has the plaintext in `$TMP_WORK/credentials.env`. Copying it to `~/.claude/credentials/credentials.env` right now skips the bootstrap-decrypt round-trip the admin would otherwise need to run on themselves (encrypt-with-passphrase → push → wait for Desktop sync → re-enter same passphrase to decrypt back to the same plaintext). Bootstrap remains the canonical path for **employees** who never see the plaintext; this step is the admin shortcut.

Run via Bash tool:

```bash
mkdir -p "$HOME/.claude/credentials"
chmod 700 "$HOME/.claude/credentials"
cp "$TMP_WORK/credentials.env" "$HOME/.claude/credentials/credentials.env"
chmod 600 "$HOME/.claude/credentials/credentials.env"

# Determine shell rc file (mirrors bootstrap.sh logic — same marker so the
# stanza isn't duplicated if the admin also runs the bootstrap later).
case "$(basename "${SHELL:-}")" in
  zsh)  RC_FILE="$HOME/.zshrc" ;;
  bash) RC_FILE="$HOME/.bashrc" ;;
  *)    RC_FILE="$([[ "$(uname -s)" == "Darwin" ]] && echo "$HOME/.zshrc" || echo "$HOME/.bashrc")" ;;
esac
touch "$RC_FILE"

CREDS_MARKER="# Palisades-Labs claude-harness-installer: credentials source"
if ! grep -Fq "$CREDS_MARKER" "$RC_FILE"; then
  {
    printf '\n%s\n' "$CREDS_MARKER"
    printf 'if [ -f "$HOME/.claude/credentials/credentials.env" ]; then\n'
    printf '  set -a; source "$HOME/.claude/credentials/credentials.env"; set +a\n'
    printf 'fi\n'
  } >> "$RC_FILE"
fi
```

The admin's plaintext credentials are now at `~/.claude/credentials/credentials.env` (chmod 600) and will load into every new shell via the rc stanza. **The plaintext value never enters Claude's context** — it's a `cp` of a temp file the skill already wrote, never echoed.

### Step 5: Ensure credentials.env is gitignored

```bash
GITIGNORE="$TMP_WORK/repo/.gitignore"
if ! grep -Fq "credentials.env" "$GITIGNORE" 2>/dev/null; then
  printf '\ncredentials.env\n' >> "$GITIGNORE"
fi
```

### Step 6: Commit and push

```bash
cd "$TMP_WORK/repo"
git add credentials/credentials.env.age .gitignore
git commit -m "chore: update credentials"
git push
```

### Step 7: Confirm distribution

Print this as the final output. **DO NOT print the passphrase** — admin already has it in their password manager.

Substitute `<CLIENT_REPO>` with the value detected in the resolution step (e.g. `lou427/insidescale-claude-harness`).

```
Credentials updated and pushed to <CLIENT_REPO>/credentials/credentials.env.age.

Your own machine is already wired up — Step 4b dropped the plaintext at ~/.claude/credentials/credentials.env (chmod 600) and added the source stanza to your shell rc. Open a new terminal, run `claude`, and the API keys are available. No bootstrap needed for you.

────────────────────────────────────────────────────────────
Next: distribute to employees
────────────────────────────────────────────────────────────

Run `/client-admin:generate-installer` to emit the decrypt-only one-liner. Distribute the one-liner + the passphrase (from your password manager) to employees via your chosen encrypted channel (1Password shared item, encrypted DM, etc.). Employees run the bootstrap because they don't have plaintext access — you do, so you skipped that step.

If rotating an existing key (same passphrase), employees pick up the new credentials.env.age at their next Claude Desktop marketplace sync — no new passphrase distribution needed.

If rotating the passphrase itself, re-run `/client-admin:generate-installer` after this skill and redistribute both the one-liner and the new passphrase; employees re-run the one-liner to decrypt under the new passphrase. Your own ~/.claude/credentials/credentials.env was already updated by Step 4b above.
```

---

## Error handling

| Scenario | Behavior |
|---|---|
| `age` not installed | Fail immediately: "age is not installed. Run: brew install age (Mac) or winget install FiloSottile.age (Windows)." |
| Wrong existing passphrase | age returns non-zero; fail with: "Wrong passphrase — the existing credentials file could not be decrypted." |
| Clone fails | Fail with: "Could not clone $CLIENT_REPO. Check that gh is authenticated and has repo access." |
| Push fails | Fail with git error; leave temp files for inspection. |

## Dependencies

- `age` — install via `brew install age` (Mac) or `winget install FiloSottile.age` (Windows)
- `git`
- `gh` — authenticated as a user with write access to the client harness repo (`gh auth status` succeeds)
