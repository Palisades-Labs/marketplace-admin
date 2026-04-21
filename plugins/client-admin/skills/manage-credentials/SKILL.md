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

**Context C — Installed harness (admin's own machine):** Read `~/.claude/settings.json` → `extraKnownMarketplaces`. For each entry, check whether `~/.claude/plugins/marketplaces/<name>/credentials/` exists — only client harness repos have this directory; marketplace catalogs do not. Exclude any entry without it. Display qualifying entries by the `name` field from their `.claude-plugin/marketplace.json` titlecased, not the repo slug. Use the single qualifying entry automatically; ask if multiple.

If none of A/B/C produces a repo, ask: "I couldn't detect a client repo. Are you running this from your master repo, from inside a client's harness repo, or on a machine with the harness installed?"

---

## Workflow

### Step 1: Clone client repo

```bash
TMP_WORK=$(mktemp -d)
trap 'rm -rf "$TMP_WORK"' EXIT
git clone --depth 1 \
  "https://x-access-token:${GITHUB_TOKEN}@github.com/${CLIENT_REPO}.git" \
  "$TMP_WORK/repo"
```

### Step 2: Load existing keys (if any)

If `$TMP_WORK/repo/credentials/credentials.env.age` exists:

```bash
# Prompt for existing passphrase to decrypt
printf "Existing credentials found. Enter the current passphrase to load and edit them: "
read -rs EXISTING_PASSPHRASE
echo ""
printf '%s\n' "$EXISTING_PASSPHRASE" | \
  age --decrypt -o "$TMP_WORK/credentials.env" "$TMP_WORK/repo/credentials/credentials.env.age"
unset EXISTING_PASSPHRASE
```

Display the current keys to the user, **masking values** (show key names, hide values):

```
Current keys:
  TAVILY_API_KEY = ****
  AVOMA_API_KEY  = ****
```

If no existing file, start with an empty credentials set.

### Step 3: Prompt for changes

Ask the user to add, update, or remove keys. Accept `KEY=value` pairs one at a time. When done, ask "Anything else? (Enter to finish)".

Write the final set to `$TMP_WORK/credentials.env` (never written to the cloned repo directory — always stays in the temp working dir).

### Step 4: Encrypt with admin-chosen passphrase

The passphrase is admin-chosen — admin picks it in their own password manager (1Password, Bitwarden, or whatever they use for secrets) BEFORE invoking this skill, then pastes it at the prompt. This keeps the passphrase lifecycle (generation, storage, rotation) entirely in the admin's toolchain.

**Security:** Per CLAUDE.md Quality & Style credentials-handling rule, `NEW_PASSPHRASE` and the plaintext credentials MUST NOT be printed, logged, included in chain-of-thought reasoning, or passed to subagents. The `read -rs` below is the only point of capture; the variables are unset after age encryption.

```bash
printf "Enter the passphrase for this credentials file (paste from your password manager; will not echo): "
read -rs NEW_PASSPHRASE
echo ""
if [ -z "$NEW_PASSPHRASE" ]; then
  echo "Passphrase cannot be empty." >&2
  exit 1
fi
printf "Confirm passphrase: "
read -rs CONFIRM_PASSPHRASE
echo ""
if [ "$NEW_PASSPHRASE" != "$CONFIRM_PASSPHRASE" ]; then
  echo "Passphrases don't match." >&2
  exit 1
fi
mkdir -p "$TMP_WORK/repo/credentials"
printf '%s\n' "$NEW_PASSPHRASE" | \
  age --encrypt --passphrase -o "$TMP_WORK/repo/credentials/credentials.env.age" "$TMP_WORK/credentials.env"
unset NEW_PASSPHRASE CONFIRM_PASSPHRASE
```

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

```
Credentials updated and pushed to <CLIENT_REPO>/credentials/credentials.env.age.

Next step: run `/client-admin:generate-installer` to emit the decrypt-only one-liner. Distribute the one-liner + the passphrase (from your password manager) to employees via your chosen encrypted channel (1Password shared item, encrypted DM, etc.).

If rotating an existing key (same passphrase), employees pick up the new credentials.env.age at their next Claude Desktop marketplace sync — no new passphrase distribution needed.

If rotating the passphrase itself, re-run `/client-admin:generate-installer` after this skill and redistribute both the one-liner and the new passphrase; employees re-run the one-liner to decrypt under the new passphrase.
```

---

## Error handling

| Scenario | Behavior |
|---|---|
| `age` not installed | Fail immediately: "age is not installed. Run: brew install age (Mac) or winget install FiloSottile.age (Windows)." |
| Wrong existing passphrase | age returns non-zero; fail with: "Wrong passphrase — the existing credentials file could not be decrypted." |
| Empty passphrase at Step 4 | Reject: "Passphrase cannot be empty." |
| Passphrase confirmation mismatch | Reject: "Passphrases don't match." |
| Clone fails | Fail with: "Could not clone $CLIENT_REPO. Check that GITHUB_TOKEN has access." |
| Push fails | Fail with git error; leave temp files for inspection. |

## Dependencies

- `age` — install via `brew install age` (Mac) or `winget install FiloSottile.age` (Windows)
- `git`
- `GITHUB_TOKEN` — must have write access to the client harness repo
