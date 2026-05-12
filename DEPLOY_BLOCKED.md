# Site Deploy Blocked — 2026-05-12

## Status

- Repo content is current: index.html and install reflect Phantom Vault v1.7.0+ (free-gift positioning, founder $82K story, `phantom edit` + guardrails + Keychain auto-unlock, new install script that ships vault-mcp).
- Most recent commit on `main`: `fc818ea`
- **Cloudflare Pages deploys have been failing since at least 2026-05-12T20:41:50Z.**
- Last successful deploy was 2026-03-01T01:59:03Z.
- Live site at https://phantomvault.riscent.com/ is still serving the v1.4.0-era content.

## What was tried (none worked)

1. `cloudflare/wrangler-action@v3` (the long-standing config that worked through March) — failed
2. Pinning to `@v3.14.1` — failed
3. Pinning `@v3.13.1` + explicit `wranglerVersion: "3.78.0"` — failed instantly (within seconds)

The fact that v3.13.1 + wranglerVersion: 3.78 failed in seconds suggests the failure happens before any deploy work — most likely an authentication error against the Cloudflare API.

## Most likely root cause

**`CLOUDFLARE_API_TOKEN` GitHub secret has expired or been revoked.**

Cloudflare API tokens can be configured with expiration dates. The token may have been created with a 60- or 90-day TTL and expired in late April or early May.

Other less likely possibilities:
- `CLOUDFLARE_ACCOUNT_ID` secret changed/wrong
- The `phantom-vault-site` Cloudflare Pages project was deleted, renamed, or its production branch changed
- Cloudflare-side rate limiting or account suspension

## Fix steps for Ryan

1. Log into Cloudflare dashboard → Pages → `phantom-vault-site` project. Confirm it exists and check its deploy history for the actual failure messages.
2. If the project is still there: go to My Profile → API Tokens. Either:
   - Regenerate the existing token, OR
   - Create a new token with scopes: `Account > Cloudflare Pages > Edit` for the right account
3. Update the GitHub repo secret: r-db/phantom-vault-site → Settings → Secrets → Actions → `CLOUDFLARE_API_TOKEN`. Paste the new token. Also verify `CLOUDFLARE_ACCOUNT_ID` matches the account showing in the Cloudflare URL bar.
4. Re-run the deploy: r-db/phantom-vault-site → Actions → "Deploy to Cloudflare Pages" → Run workflow → main → Run.

(I added `workflow_dispatch:` to the trigger list so the manual re-run button is available.)

## What's not blocked

- The phantom-vault product itself is live and working. Latest release: **v1.7.1** at https://github.com/r-db/phantom-vault/releases/tag/v1.7.1
- A new user can still install today by running the install.sh from the GitHub raw URL directly:
  ```
  curl -fsSL https://raw.githubusercontent.com/r-db/phantom-vault/main/install.sh | bash
  ```
  This works perfectly. They just can't get there via the website link until Cloudflare is unblocked.

## Diagnostic notes for next-session Claude

- Run logs are gated behind GitHub auth. Either `gh auth login` first, or fetch with `curl -H "Authorization: token $GH_TOKEN" .../runs/{id}/logs` and unzip.
- The fact that v3.13.1 fails fast (seconds) is the strongest signal that auth is the cause; deploy issues take longer to surface.
