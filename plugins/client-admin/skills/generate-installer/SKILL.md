---
name: generate-installer
description: Produce a pre-composed install command (Mac bash + Windows PowerShell one-liners) for a client admin to distribute to their team. The one-liner decrypts the shared credentials from the Claude Desktop marketplace sync. Use when the admin is ready to onboard employees. Never run this as an employee — the admin will send you a command.
---

*Last Edited: 2026-04-20*

# generate-installer

**Execute this skill directly. Do not enter plan mode. Do not expose detection logic, context rules, or internal derivation steps to the user.** The user should see only: what you found, what you need from them, and the finished result.

## Invocation

```
/client-admin:generate-installer
/client-admin:generate-installer <slug>      # master-repo-only: shortcut when you know which client
```

## When to use

- You are the **client admin** who has finished `/client-admin:manage-credentials` (encrypted credentials pushed; passphrase in hand) and are ready to distribute install commands to your team.
- OR you are the **consultant** prepping commands inside the master repo on behalf of a client admin.

## When NOT to use

- You are a client employee. The admin will send you the install command — do not try to generate it yourself.
- You have not yet run `/client-admin:manage-credentials`. Do that first: it encrypts `credentials.env.age`, pushes it to the harness repo, and gives you the passphrase that employees will need. The install command this skill emits is useless without that file synced through Claude Desktop.

<!-- ============================================================
     INTERNAL LOGIC — do not surface any of this to the user.
     Execute silently; report only the result.
     ============================================================ -->

## Detection logic (internal — do not surface to the user)

### Pre-flight admin/consultant gate — RUN THIS FIRST

This skill emits install commands for the entire team. Its intended audience is **client admins** and **consultants** (coaching admins through the flow). Employees typing `/client-admin:generate-installer` out of curiosity is still a risk — the install block they'd get back references credentials the admin hasn't authorized them to distribute. Keep the gate.

Post-migration, the programmatic admin signals (`gh auth`, `GITHUB_TOKEN` rc-marker) are gone — nothing on an admin's machine is guaranteed to look different from an employee's. `clients.json` is the only automatic signal that survives. Admins typically run this from their own Claude Code session without `clients.json`, so the skill falls back to conversational confirmation. Single question keeps employee-refusal intact without blocking admins.

```bash
# Post-migration signals:
#   - clients.json in $PWD                              → consultant (certain)
#   - .claude-plugin/marketplace.json in $PWD           → admin OR consultant
#     (admin working inside their harness checkout, or consultant inside a client repo)
#   - Neither present                                    → unknown — ask the user
if [ -f clients.json ]; then
  ROLE=consultant
elif [ -f .claude-plugin/marketplace.json ]; then
  ROLE=admin_or_consultant
else
  ROLE=unknown
fi
```

If `ROLE=unknown`, ask the user ONE question (no multi-step triage):

> Quick check before I generate anything — who's running this: **admin**, **consultant**, or **employee**?

- "admin" or "consultant" → proceed to detection below.
- "employee" (or any equivalent — "team member", "I'm just using the harness", etc.) → refuse with the message below and stop.
- "not sure" → ask them to clarify; if still unclear, default to refuse.

Employee refusal text:

> This skill is for the admin who distributes the harness, not for team members who use it. It emits install commands that reference encrypted credentials employees aren't authorized to redistribute — your admin issues those.
>
> If you're trying to install the harness on your own machine, ask your admin to send you the install command (they'll run `/client-admin:generate-installer` themselves and share the result).

Otherwise continue to the detection below.

### Resolve target repo

Try these context rules in order; the first that succeeds wins.

**Context A — Master repo (consultant):** `clients.json` exists in `$PWD`.

```bash
if [ -f clients.json ]; then
  jq -r '.clients[] | "\(.name)\t\(.repo)\t\(.status)"' clients.json
fi
```

If a `slug` arg was given, match it against `.name`; else ask the user which client. Display client names titlecased (e.g., `test-client` → "Test Client"), not raw slugs. Capture `.repo`.

**Context B — Client repo (admin with the harness checked out):** `.claude-plugin/marketplace.json` exists in `$PWD`.

```bash
if [ -f .claude-plugin/marketplace.json ]; then
  jq -r '.name' .claude-plugin/marketplace.json
  git config --get remote.origin.url
fi
```

Extract `<org>/<repo>` from the remote URL (strip `https://github.com/` prefix and `.git` suffix).

**Context C — Installed harness (admin's own machine):** Read `~/.claude/plugins/known_marketplaces.json` (the runtime registry — more reliable than `settings.json` which may be empty after a UI-based add).

```bash
jq -r 'to_entries[] | "\(.key)\t\(.value.source.url)"' ~/.claude/plugins/known_marketplaces.json
```

For each entry, check whether `~/.claude/plugins/marketplaces/<name>/credentials/` exists — only client harness repos have this directory; marketplace catalogs do not. Exclude entries without it. Display qualifying entries by the `name` field from `.claude-plugin/marketplace.json` titlecased, not the repo slug. If exactly one qualifies, use it automatically. If multiple, ask the user which.

If none of A/B/C produce a repo, tell the user: "I couldn't detect a client repo. Are you running this from your master repo, from inside a client's harness repo, or on a machine with the harness installed?"

### Derive marketplace name

```bash
CLIENT_REPO="<org>/<repo>"              # from above
REPO_NAME="${CLIENT_REPO##*/}"
MARKETPLACE_NAME="${REPO_NAME%-claude-harness}"
```

Must match what `bootstrap.sh` / `bootstrap.ps1` / `/onboard-client` derive. If Context B surfaced a `marketplace.json` name that differs, the repo was onboarded before the Option C migration — flag it to the user and use the derived value anyway.

<!-- ============================================================
     USER-FACING WORKFLOW — this is what the user sees.
     ============================================================ -->

## What the user sees

Execute these steps in order. Each step is a beat in the conversation — tell the user what's happening and what you need.

### Step 1: Confirm the target

After running the detection logic above, tell the user what you found:

> Generating install commands for **[CLIENT_NAME]** (`org/repo`).

If detection failed or found multiple clients, ask the user to clarify. Keep it to one question.

### Step 2: Confirm credentials + passphrase mode

Credentials are the only secret this skill touches. Before emitting anything, confirm the admin has actually encrypted + pushed them:

> Have you run `/client-admin:manage-credentials` yet? It encrypts `credentials.env.age`, pushes it to the harness repo (so Claude Desktop can sync it to employee machines), and gives you the passphrase to send employees separately.

If they haven't run it, stop here and prompt them to do so before continuing.

If they have, ask about passphrase mode. Two options, default to placeholder mode (per `palisades-labs-harness.md` § "Minimizing LLM exposure of client credentials"):

> How do you want to handle the passphrase in the output?
>
> **Placeholder mode (default, minimizes LLM exposure).** I emit the passphrase block with a literal `<PASSPHRASE>` marker. You substitute the real value in the outgoing message where you're about to send it (Slack DM, 1Password note, in-person). The passphrase never enters this chat. Recommended for real client distributions.
>
> **Bake mode (convenience).** Paste the passphrase here and I bake it into the passphrase block, ready to forward. The passphrase appears in my context and the output, which may be retained by Anthropic's infrastructure (prompt cache, API logs). Acceptable when you own every involved principal (e.g., Aaron running a drill against his own test repo) or when the conversation is explicitly ephemeral.
>
> Which mode? (default: placeholder)

Wait for the user's choice. Hold the passphrase value (bake mode) or the `<PASSPHRASE>` marker (placeholder mode) for Step 3.

### Step 3: Emit the install message

Substitute `<CLIENT_NAME>` and `<ORG>/<REPO>` always. For `<PASSPHRASE>`: substitute the literal value in **bake mode**; leave the literal `<PASSPHRASE>` marker in **placeholder mode**.

**Frame the output for the admin first.** The admin needs to know the block below is what they forward to their team — NOT what they run themselves. Lead with one short framing sentence addressed to the admin, then output the forwardable block. Without the framing, admins read the recipient-facing "To install the <CLIENT_NAME>…" line and get confused about whose instructions they're looking at.

**If placeholder mode was chosen**, add an extra admin-facing line above the passphrase block: *"Before sending, replace `<PASSPHRASE>` with the real value — substitute in the draft of your outgoing message, not back into this chat."*

Output format — admin framing first, then the forwardable install block, then the passphrase block:

> For your team to install the **<CLIENT_NAME>** Claude Code harness, send them the message below — it contains both Mac and Windows instructions, so each team member uses whichever matches their machine:

```
To install the <CLIENT_NAME> Claude Code harness:

**Before running this one-liner:** Add the `<ORG>/<REPO>` marketplace in Claude Desktop FIRST and wait for sync to complete. The encrypted credentials file is sourced from the marketplace sync directory — if the marketplace isn't synced yet, install will fail with a clear "marketplace not synced yet" error.

• Mac / Linux — open Terminal, paste the line below, press Enter:

    bash <(curl -fsSL https://raw.githubusercontent.com/Palisades-Labs/claude-harness-installer/main/bootstrap.sh) --decrypt <ORG>/<REPO>

• Windows — open PowerShell. Copy ALL lines below, paste them together, press Enter:

    $env:DECRYPT_MODE='1'
    $env:CLIENT_REPO='<ORG>/<REPO>'
    Set-ExecutionPolicy -Scope Process Bypass -Force
    iwr -useb https://raw.githubusercontent.com/Palisades-Labs/claude-harness-installer/main/bootstrap.ps1 | iex

After install, open a NEW terminal (or PowerShell) and run `claude`. Desktop auto-syncs future tool updates — no manual sync command needed.
```

Then, immediately after the install block, add a **separate passphrase block** for the admin to send through a different channel:

> Bootstrap will prompt each employee for a setup passphrase during install. Send them the passphrase below **separately** — not in the same message as the install command (a Slack DM, in-person, or any private channel):
>
> `<PASSPHRASE>`
>
> They type or paste it when prompted. It is not echoed and does not appear in their shell history.

**Paste-robustness rule (critical):** The Mac/Linux form is a single statement — `bash <(curl ...) --decrypt <org>/<repo>` has no inline env vars, so visual wrapping in Slack/email doesn't split interdependent lines. The Windows form still has three setup statements plus the `iwr|iex` call; keep each on its own line — do NOT collapse them into a single long line. Chat and email clients soft-wrap long commands and users' paste behavior often captures the visual wrap as real newlines, splitting the command into broken fragments. The multi-statement form is robust to wrap because each statement stands alone. **Do not shorten the PowerShell block to a one-liner under any circumstance.**

### Step 4: Rotation guidance

After the install block, print:

```
API key rotation
----------------
To add a new API key or rotate an existing one (Tavily, Avoma, or any other), run `/client-admin:manage-credentials`, update the key, and push. Employees pick up the new key on their next Claude Desktop marketplace sync — no new install command or passphrase distribution needed (as long as you keep the same passphrase).

If you rotate the passphrase itself, re-run `/client-admin:generate-installer` and redistribute both the install command and the new passphrase; employees re-run the one-liner to decrypt under the new passphrase.
```

## Cross-layer notes

- The emitted one-liners must stay in sync with `bootstrap.sh` and `bootstrap.ps1` arg parsing in the `claude-harness-installer` repo. The current contract:
  - `bootstrap.sh --decrypt <org>/<repo>`
  - `bootstrap.ps1 -Decrypt -Repo <org>/<repo>` (locally) OR `$env:DECRYPT_MODE='1'; $env:CLIENT_REPO='<org>/<repo>'; iwr|iex` (one-liner form; param() binding is skipped under iex so the script reads `$env:DECRYPT_MODE` and `$env:CLIENT_REPO` as fallbacks).
  When either bootstrap changes its invocation, update the template above.
- Option C naming invariant lives in four places: this skill, `bootstrap.sh`, `bootstrap.ps1`, and `/onboard-client`. All four must derive marketplace names identically (`basename(repo)` with any trailing `-claude-harness` stripped).

## Dependencies

- `jq` — used on the operator's machine to parse `clients.json` and `~/.claude/settings.json` during detection. Post-migration, `bootstrap.sh` / `bootstrap.ps1` no longer install prereqs (they're decrypt-only), so `jq` must come from base tooling.
