# Admin Setup Guide — Palisades-Labs Claude Code Harness

This guide walks you through everything you need to do as the admin who manages your team's Claude Code harness. By the end you'll have:

1. The admin tools installed on your own machine.
2. Your team's API keys (Tavily, Avoma, etc.) encrypted and pushed to your harness repo.
3. An install command ready to send to your team, plus a passphrase to send through a separate secure channel.

If anything in this guide is unclear, ask Aaron — that's a doc bug, not a you problem.

---

## Before you start — what you need

Make sure all of these are installed on your machine before you begin. If you're missing any, install them first.

- **Claude Code** — install from https://docs.claude.com/en/docs/claude-code/setup
- **Claude Desktop** — install from https://claude.com/download
- **A GitHub account** with read+write access to your company's harness repo (the consultant set this up during onboarding; if you're not sure, ask)
- **`gh` (GitHub CLI)** — install from https://cli.github.com — and sign in via `gh auth login` once. We use this so the admin tools can push encrypted credentials to your repo without dealing with tokens.
- **A password manager** (1Password recommended; Bitwarden also fine). You will save the encryption passphrase here.

You do **not** need Homebrew. The harness installer installs `age` (the encryption tool) automatically when needed.

---

## Step 1 — Install the admin tools in Claude Code

Open Claude Code in your terminal. If you've never used it before:

1. Open Terminal (Mac) or PowerShell (Windows)
2. Type `claude` and press Enter
3. Wait for the welcome screen to load

Once Claude Code is running, you'll see a prompt at the bottom of the terminal where you type messages.

**Add the admin marketplace.** Type this exactly and press Enter:

```
/plugin marketplace add Palisades-Labs/marketplace-admin
```

Wait until you see "marketplace added" or similar confirmation.

**Install the admin plugin.** Type this exactly and press Enter:

```
/plugin install client-admin@palisades-admin
```

Wait until you see "installed" confirmation. To verify it worked, type `/plugin list` and press Enter — you should see `client-admin` in the list.

You now have two new commands available: `/client-admin:manage-credentials` and `/client-admin:generate-installer`.

---

## Step 2 — Set up your team's encrypted credentials

This is where you tell Claude what API keys your team needs (Tavily for web search, Avoma for meeting notes, whatever applies). You'll encrypt them with a passphrase and push them to your team's harness repo. Your team will use that same passphrase later to decrypt the file on their own machines.

**Pick a passphrase first.** Open your password manager (1Password, Bitwarden, etc.). Create a new item called `<your-company>-harness-passphrase`. Generate a strong passphrase (at least 20 characters). Copy it to the clipboard. You'll paste it in Step 2 below — DO NOT type it into the Claude Code chat.

**Run the credentials setup.** In Claude Code, type:

```
/client-admin:manage-credentials
```

Press Enter. The skill will:

1. **Detect your harness repo** by looking at your installed marketplaces. If you have only one, it'll auto-pick. If multiple, it'll ask which.

2. **Check for an existing credentials file.** If this is your first time, the file doesn't exist yet — Claude will say so and move on. If a previous file exists, Claude will ask:

   > "I found an existing credentials file. Do you know the current passphrase?
   >  1. Yes — enter it now to load and review the current keys before editing
   >  2. No / Start fresh — replace the file with new credentials"

   For your first setup, choose option 2 (Start fresh). For later updates where you want to add a new key without losing existing ones, choose option 1.

3. **Ask which API keys to add.** For each key (e.g., `TAVILY_API_KEY`, `AVOMA_API_KEY`), Claude will give you a copy-paste command in a code block, like:

   ```bash
   printf "TAVILY_API_KEY: " && read -rs v && echo && echo "TAVILY_API_KEY=$v" >> /tmp/.../credentials.env
   ```

   **Open a NEW terminal window** (Cmd+T on Mac, or open a second PowerShell). Paste the command. Press Enter. The terminal will print `TAVILY_API_KEY:` and wait. Paste the actual API key value — it will NOT show on screen as you paste, that's intentional. Press Enter.

   Repeat for each key. When you're done, type `done` in the Claude Code window and press Enter.

4. **Encrypt with your passphrase.** Claude will run the encryption tool (`age`). At the terminal you'll see:

   ```
   Enter passphrase (leave empty to autogenerate a secure one):
   ```

   Paste the passphrase you saved in your password manager (Cmd+V on Mac, Ctrl+V on Windows). Nothing appears on screen as you paste — that's correct. Press Enter.

   You'll see another prompt:

   ```
   Confirm passphrase:
   ```

   Paste the same passphrase again. Press Enter.

5. **Push to GitHub.** Claude will commit the encrypted file (`credentials/credentials.env.age`) to your harness repo and push.

6. **Confirmation.** You'll see "Credentials updated and pushed" or similar.

The plaintext API keys never appear in Claude's chat, never go to GitHub, and never appear in any log. Only the encrypted file is pushed.

---

## Step 3 — Generate the install command for your team

Now produce the install message you'll send to your team.

In Claude Code, type:

```
/client-admin:generate-installer
```

Press Enter. The skill will:

1. **Detect your harness repo** (same as Step 2).

2. **Confirm credentials are set up.** It'll ask if you've already run `/client-admin:manage-credentials`. Say yes.

3. **Ask how you'll send the passphrase to your team.** Options:

   - **1. 1Password, Bitwarden, or another secure channel (recommended)** — leaves a `<PASSPHRASE>` placeholder in the output that you fill in before sending.
   - **2. Slack direct message** — same as above but with a heads-up that Slack DMs are stored on Slack's servers.
   - **3. Email** — same as above but flagging that email is stored in plaintext on mail servers.
   - **4. Paste it here (I'll bake it into the output)** — only use this for internal drills against your own test repo. The passphrase will appear in this Claude conversation.

   For real client distributions, choose option 1.

4. **Print the install message as four separate copy blocks** plus a passphrase block. Each block has its own copy button.

---

## Step 4 — Send it to your team

You're going to compose ONE message to your team that contains the install instructions. Then you'll send the passphrase SEPARATELY through your password manager.

**Compose the install message.** Open Slack (or email, or whatever channel your team uses). Start a new message addressed to your team or the appropriate channel. Then:

1. Copy the **intro block** from Claude's output (click the copy button on that code block). Paste it at the top of your message.

2. Copy the **Mac / Linux command block**. Paste it under a heading like "**Mac / Linux:**".

3. Copy the **Windows command block**. Paste it under a heading like "**Windows:**". IMPORTANT: do not modify the four lines or collapse them — they need to stay as four separate lines for paste-robustness.

4. Copy the **after-install block**. Paste it at the end.

5. **Replace `<PASSPHRASE>` placeholder** if it appears anywhere in your message — but actually you should NOT put the passphrase in this message at all. Delete the entire passphrase block from this message; it goes in a separate message.

6. Send the message.

**Send the passphrase separately.** Now do one of:

- **1Password / Bitwarden share:** create a shared item with your team called `<your-company>-harness-passphrase`. Paste the passphrase value. Share with team members who need it. They'll see it in their own password manager.
- **Slack DM:** message each team member directly (NOT in a channel). Paste the passphrase. Tell them to delete the message after they've used it. (Less ideal because Slack stores DMs.)
- **In-person:** if everyone is in the office, just tell them.

The passphrase MUST go through a different channel than the install command. If both are in the same Slack message and that message is forwarded or screenshotted, the harness is compromised.

---

## Step 5 — Adding or rotating an API key later

API key changes are easy. From Claude Code:

```
/client-admin:manage-credentials
```

When asked about the existing credentials file, choose **option 1 (Yes)** this time. You'll be prompted for the current passphrase — paste it from your password manager.

Add or update the key the same way as Step 2.3. When asked for the encryption passphrase at the end, paste the SAME passphrase as before. Your team won't need to do anything new — Claude Desktop auto-syncs the updated encrypted file, and the next time they open a new terminal (or re-run the bootstrap), they pick up the new values.

If you want to rotate the passphrase itself (security event, employee left the company, etc.), choose a new passphrase and then re-run `/client-admin:generate-installer` and redistribute both the install command and the new passphrase. Team members re-run the install command and enter the new passphrase.

---

## Common problems

**"age is not installed"** — The bootstrap auto-installs `age` now. If you see this error, your script is out of date. Ask Aaron to verify.

**"no coprocess"** — You're on zsh and an older version of the skill emitted a bash-specific command. The latest skill version handles this. Ask Aaron to check that the marketplace is up to date.

**"marketplace not synced yet"** — Wait 30 seconds for Claude Desktop to finish syncing your marketplace, then retry. If it still fails after 2 minutes, remove and re-add the marketplace in Claude Desktop.

**"credentials.env.age not found"** — The encrypted file lives in a `credentials/` subdirectory. The latest bootstrap looks there correctly. If you see this error, the bootstrap script the team is running is stale — see CDN issue below.

**"Wrong passphrase — could not be decrypted"** — Check that the passphrase you're entering matches what's in your password manager exactly (no extra spaces, no extra characters from copy-paste). If you've forgotten it, you'll need to start fresh with a new passphrase and redistribute (which means asking your team to re-run their install).

**Your team is getting old script behavior after you pushed an update** — `raw.githubusercontent.com` (where the bootstrap script is served from) has a CDN that can lag 5–30+ minutes after a push. Wait it out, or ask Aaron to verify.

**Cache problem after updating a skill** — If you push a skill update and Claude on your team's machines still uses the old version, they may need to clear `~/.claude/plugins/cache/<marketplace-name>/` and restart Claude Code. Aaron can walk anyone through this.

**You ran a command and got refused with "this is for admins"** — You're probably running it from a Claude Code session that doesn't have a `.claude-plugin/marketplace.json` in the working directory. Tell Claude "I'm the admin" when it asks.

For deeper troubleshooting beyond what's covered above, contact the harness maintainer at Palisades Labs.
