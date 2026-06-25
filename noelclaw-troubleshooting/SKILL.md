---
name: noelclaw-troubleshooting
description: Debug common NoelClaw MCP errors - tools not appearing, old version loading, web_search fails, schedule_research fails, swap refused, rate limit 429, Convex connection issues, Buffer polyfill errors. Includes noelclaw doctor health check.
tags: [noelclaw, troubleshooting, debugging, mcp, errors, health-check]
---

# NoelClaw Troubleshooting

## Overview

Common errors when running `@noelclaw/mcp@3.30.0` and their fixes. Start with `noelclaw doctor` for a health check, then match symptoms below.

**Pinned install (never use `@latest` — supply-chain risk):**
```bash
npx -y @noelclaw/mcp@3.30.0
```

---

## 60-Second Quickstart

```bash
noelclaw doctor          # Check all systems
noelclaw status          # Quick auth + version check
npx clear-npx-cache      # If old version cached
```

**Expected output:**
- `doctor` prints ✅/❌ per subsystem — expect **0 critical ❌ errors**
- `status` prints a one-line summary — expect **version `3.30.0`** and `auth: ✓`
- If version is wrong (e.g., shows `3.29.0` or older), run `npx clear-npx-cache` then re-run `doctor` — version should now read `3.30.0`

---

## Health Check

### `noelclaw doctor`

Run this first. It checks:

- MCP server connectivity
- Session token validity (`NOELCLAW_SESSION_TOKEN`)
- Convex backend reachability
- Required API keys presence (not values — just set/unset)
- Wallet configuration
- Trigger.dev connectivity (if `TRIGGER_SECRET_KEY` is set)

```bash
npx -y @noelclaw/mcp@3.30.0 doctor
```

Output shows ✅/❌ for each check. Fix all ❌ items before proceeding.

---

## Common Errors

### 1. Tools Not Appearing in MCP Client

**Symptom:** MCP client (Claude Desktop, Cursor, etc.) shows 0 or partial tools from NoelClaw.

**Cause:** MCP client cached an old connection or failed to restart after config changes.

**Fix:**

1. Verify the MCP config JSON is valid:
   ```bash
   # Check config syntax
   cat ~/.config/claude/claude_desktop_config.json | python3 -m json.tool
   ```
2. **Restart the MCP client** completely (not just reload — quit and reopen)
3. Check the MCP client logs for connection errors:
   - Claude Desktop: `~/Library/Logs/Claude/mcp.log` (macOS) or `%APPDATA%\Claude\logs\mcp.log` (Windows)
   - Cursor: Check Output panel → select "MCP"
4. Verify the server starts manually:
   ```bash
   npx -y @noelclaw/mcp@3.30.0 --help
   ```

### 2. Old Version Loading (npx Cache)

**Symptom:** Tools from an older `@noelclaw/mcp` version appear even after updating the pin. New tools are missing.

**Cause:** npx caches packages and may serve a stale cached version.

**Fix:**

```bash
# Clear npx cache
npx clear-npx-cache

# Or manually clear the npx cache directory
# Windows (git-bash):
rm -rf ~/AppData/Local/npm-cache/_npx
# macOS/Linux:
rm -rf ~/.npm/_npx

# Re-run with explicit pin
npx -y @noelclaw/mcp@3.30.0 doctor
```

Verify the running version:
```bash
npx -y @noelclaw/mcp@3.30.0 --version
# Should output: 3.30.0
```

### 3. `web_search` Fails

**Symptom:** `web_search` returns an error like `FIRECRAWL_API_KEY not set` or `Web search unavailable`.

**Cause:** `FIRECRAWL_API_KEY` is not set in the MCP client environment.

**Fix:**

1. Get a Firecrawl API key from [firecrawl.dev](https://firecrawl.dev)
2. Add it to the MCP client config `env` block:
   ```json
   {
     "mcpServers": {
       "noelclaw": {
         "command": "npx",
         "args": ["-y", "@noelclaw/mcp@3.30.0"],
         "env": {
           "NOELCLAW_SESSION_TOKEN": "noel_...",
           "FIRECRAWL_API_KEY": "fc-..."
         }
       }
     }
   }
   ```
3. **Restart the MCP client** after saving the config
4. Verify:
   ```bash
   npx -y @noelclaw/mcp@3.30.0 doctor
   # FIRECRAWL_API_KEY should show ✅
   ```

### 4. `schedule_research` or Monitors Fail

**Symptom:** `schedule_research`, `create_monitor`, or any server-side scheduling tool returns `TRIGGER_SECRET_KEY not set` or `Trigger.dev connection failed`.

**Cause:** Trigger.dev credentials are missing or invalid. Server-side jobs (monitors, research shifts) run on Trigger.dev infrastructure, not locally.

**Fix:**

1. Create a Trigger.dev project at [trigger.dev](https://trigger.dev)
2. Get `TRIGGER_SECRET_KEY` and `TRIGGER_PROJECT_REF` from the project dashboard
3. Set them in the worker environment:
   ```bash
   cd C:\Users\sagir\Downloads\noelclaw\noelapp\worker
   # Edit .env:
   # TRIGGER_SECRET_KEY=tr_dev_...
   # TRIGGER_PROJECT_REF=proj_...
   ```
4. Deploy the worker:
   ```bash
   cd C:\Users\sagir\Downloads\noelclaw\noelapp\worker
   npm run deploy
   ```
5. Verify connectivity:
   ```bash
   npx -y @noelclaw/mcp@3.30.0 doctor
   # Trigger.dev should show ✅
   ```

> **Security note:** Creating server-side monitors requires explicit user confirmation. Monitors continue running after the MCP process exits and may incur LLM/API costs. See Security Boundary 6 in the `noelclaw` skill.

### 5. Swap Refused (Price Impact Cap)

**Symptom:** `swap_tokens` returns an error like `Price impact exceeds threshold` or `Swap rejected: slippage too high`.

**Cause:** The swap's price impact exceeds the safety threshold. This is a **protective measure**, not a bug.

**Fix:**

1. Call `estimate_swap` first to preview the expected output and price impact:
   ```bash
   npx -y @noelclaw/mcp@3.30.0 estimate_swap \
     --fromToken USDC \
     --toToken ETH \
     --amount 50000000
   ```
2. Review the output — check `priceImpact` percentage and `expectedOutput`
3. If price impact is too high:
   - **Reduce the swap amount** — large swaps move the market more
   - **Try a different token pair** — low-liquidity pairs have higher impact
   - **Split into multiple smaller swaps** — execute over time
4. Only execute `swap_tokens` after reviewing the estimate and confirming with the user

> **Security note:** Mainnet swaps require estimate → preview → confirm → execute flow. See Security Boundary 2 in the `noelclaw` skill. Never execute without explicit user confirmation.

### 6. Rate Limit 429

**Symptom:** API calls return HTTP 429 (Too Many Requests). Common with Bankr, CoinGecko, or Firecrawl.

**Cause:** API rate limit exceeded.

**Fix:**

- **Auto-retry is built-in** — most NoelClaw API calls have automatic retry with exponential backoff. If a 429 still surfaces, the retry budget was exhausted.
- **Wait and retry** — rate limits typically reset within 60 seconds
- **Reduce frequency** — if running automations, increase `intervalMinutes`
- **Check CoinGecko specifically** — the automation engine batches all price fetches into one request. If you have many price-monitored automations, you may hit CoinGecko's free-tier limit (10-30 req/min). Consolidate or upgrade.
- **Bankr timeout** — `callBankr` polls every 2s for up to 60 polls (120s). Long-running Bankr jobs time out. If this happens, the job may still complete server-side — check the result later.

### 7. Convex Connection Issues

**Symptom:** Tools return `Convex connection error`, `WebSocket closed`, or `Function execution failed`.

**Cause:** Convex backend is unreachable or misconfigured.

**Fix:**

1. Check Convex deployment status:
   ```bash
   cd C:\Users\sagir\Downloads\noelclaw\noelapp\app
   npx convex dashboard
   ```
2. Verify `VITE_CONVEX_URL` is set and the deployment is active
3. Check Convex status page for outages
4. If developing locally, ensure `npx convex dev` is running:
   ```bash
   cd C:\Users\sagir\Downloads\noelclaw\noelapp\app
   npx convex dev
   ```
5. Verify Convex environment variables are set in the dashboard (not just `.env.local`):
   - `BANKR_API_KEY`, `WALLET_ENCRYPTION_KEY`, `ZX_API_KEY`, `ALCHEMY_API_KEY`
6. Re-run health check:
   ```bash
   npx -y @noelclaw/mcp@3.30.0 doctor
   ```

### 8. Buffer Polyfill Errors

**Symptom:** `ReferenceError: Buffer is not defined` or `process is not defined` when running the MCP server.

**Cause:** The MCP server uses Node.js built-ins (`Buffer`, `process`) that aren't available in browser-like environments or certain bundler configurations.

**Fix:**

1. **If running via MCP client (Claude Desktop, Cursor):** The client should spawn Node.js directly via `npx`. Ensure the `command` is `npx` (not a browser-based runner).
2. **If running in a bundler environment:** Add polyfills:
   ```bash
   npm install --save-dev buffer process
   ```
   Then in your entry point:
   ```javascript
   import { Buffer } from 'buffer';
   globalThis.Buffer = globalThis.Buffer || Buffer;
   ```
3. **If running via `npx` and still failing:** Ensure Node.js >= 18:
   ```bash
   node --version  # must be >= 18
   ```
4. **If using a custom Node.js path:** Make sure `npx` resolves to the correct Node version:
   ```bash
   which npx
   npx -y @noelclaw/mcp@3.30.0 doctor
   ```

### 8. Agents Respond with "Local Mode" Messages

**Symptom:** Chat responses say "Running in local mode — connect an LLM API key in Settings for full AI responses" instead of real AI responses.

**Cause:** No LLM API key is configured in the Convex dashboard. The chat backend (`convex/chat.ts`) has a multi-provider system that tries: Bankr → OpenAI → Anthropic → Groq → OpenRouter → Custom. If none are set, it falls back to rule-based local responses.

**Fix:**
1. Get an API key from any provider (Groq is free — [groq.com](https://groq.com))
2. Set the env var in Convex dashboard: `GROQ_API_KEY=gsk_...`
3. Optionally set `GROQ_MODEL=llama-3.3-70b-versatile`
4. Restart `npx convex dev`
5. Verify: chat with an agent — responses should now be full AI

**Alternative providers:**
- `OPENAI_API_KEY` + `OPENAI_MODEL=gpt-4o-mini`
- `ANTHROPIC_API_KEY` + `ANTHROPIC_MODEL=claude-sonnet-4-20250514`
- `OPENROUTER_KEY` + `OPENROUTER_MODEL=meta-llama/llama-3.3-70b-instruct:free`
- `CUSTOM_LLM_ENDPOINT` + `CUSTOM_LLM_KEY` + `CUSTOM_LLM_MODEL` (any OpenAI-compatible)

### 9. Convex Deploy: `internalAction is not defined`

**Symptom:** `npx convex dev` fails with `Failed to analyze <file>.js: internalAction is not defined`.

**Cause:** A Convex file uses `internalAction()` but the import only includes `action`. TypeScript doesn't catch this — it resolves types from generated definitions that export both. The error only surfaces at Convex push time.

**Fix:**
```typescript
// BAD — runtime error
import { action } from "./_generated/server";

// GOOD
import { action, internalAction } from "./_generated/server";
```

Then re-run `npx convex dev`. This happens when security-hardening converts `action()` → `internalAction()` but forgets the import. See `noelclaw-webapp-dev` skill → `references/convex-security-patterns.md`.

### 10. Convex Deploy: `Property 'createWallet' does not exist on type`

**Symptom:** `npx convex dev` fails TypeScript typecheck with `Property 'X' does not exist on type`.

**Cause:** When migrating `api.module.function` → `internal.module.function` (for security), `replace_all=true` patching can miss callers with different argument shapes. Example: `{ userId }` matches but `{ userId: session.userId }` doesn't.

**Fix:**
```bash
# Find all missed callers
grep -rn "api.walletActions.createWallet" convex/

# Fix each individually — change api. to internal.
# Then re-run:
npx convex dev
```

Also check: when converting `query` → `mutation`, callers must change `ctx.runQuery` → `ctx.runMutation`.

### 11. Vite Error: `Expected "}" but found "You"`

**Symptom:** Vite browser overlay shows `Expected "}" but found "You"` at a specific line in a `.ts` file.

**Cause:** A template literal (backtick string) is missing its closing backtick. This happens when a subagent patches a multi-line `systemPrompt` string and loses track of the closing `` ` ``.

**Fix:**
1. Read the file around the error line
2. Find the opening backtick (usually `systemPrompt: \`You are...`)
3. Add the closing backtick before the next property (e.g., before `voiceId:`)
4. Run `npx tsc --noEmit` to verify
5. Run `npx convex dev` to confirm Convex accepts it

### 12. Convex `[WARN]` Messages During Deploy

**Symptom:** `npx convex dev` prints many `[WARN]` messages about line numbers and export regex.

**Cause:** Convex's module analysis produces warnings for minified/bundled code. These are informational only.

**Fix:** Ignore them. Only `Error:` lines block deployment. Don't chase warnings.

---

## Diagnostic Command Sequence

When something is wrong and you're not sure what, run these in order:

```bash
# 1. Check version is correct (not cached old version)
npx -y @noelclaw/mcp@3.30.0 --version
# Expected: 3.29.0

# 2. Run health check
npx -y @noelclaw/mcp@3.30.0 doctor
# Fix all ❌ items

# 3. Clear cache if version is wrong
npx clear-npx-cache

# 4. Verify env vars are set
echo $NOELCLAW_SESSION_TOKEN  # should be non-empty
echo $FIRECRAWL_API_KEY       # should be non-empty for web_search

# 5. Check Node version
node --version  # >= 18

# 6. Test basic vault round-trip
npx -y @noelclaw/mcp@3.30.0 vault_save --title "test" --content "diagnostic test"
npx -y @noelclaw/mcp@3.30.0 vault_search --query "diagnostic" --limit 1
```

---

## Related Skills

- `noelclaw` — Main technical reference (103 tools, security boundaries, config vars)
- `noelclaw-dev-setup` — Full monorepo setup, env vars, local dev workflow
- `noelclaw-defi` — DeFi workflows (swap errors, wallet issues)
- `noelclaw-automation` — Automation engine, cron, research shifts
- `noelclaw-webapp-dev` — Multi-provider chat system, Convex security patterns
- `noelclaw-webapp-dev` — Webapp dev (includes `references/convex-security-patterns.md` for Convex security audit checklist)

---

## Pitfalls

- **Always restart the MCP client after config changes** — most "tools not appearing" issues are stale connections, not real errors.
- **npx cache is persistent** — clearing it with `npx clear-npx-cache` is the only reliable way to force a fresh download.
- **Convex env vars are NOT in `.env.local`** — they're set in the Convex dashboard. Local `.env.local` is for Vite/frontend only.
- **429 errors may have auto-retry** — if you see a 429 in logs, check if the operation eventually succeeded before reporting failure.
- **Price impact cap is a feature** — don't try to bypass it. Reduce the swap amount instead.
- **`doctor` checks presence, not validity** — it confirms keys are set, not that they work. A key can be present but expired/invalid.
- **Buffer errors mean wrong runtime** — the MCP server needs Node.js, not a browser environment. Check how the MCP client spawns the server.

---

## Verification

Run these checks and verify the **expected output** matches:

1. **Health check:** `noelclaw doctor`
   - ✅ Expected: Prints ✅/❌ per subsystem. **0 critical ❌ errors.** All of session token, Convex, API keys, Node.js should show ✅. Minor warnings (e.g., optional API key unset) are acceptable.
2. **Version check:** `npx -y @noelclaw/mcp@3.30.0 --version`
   - ✅ Expected: Prints **`3.30.0`** exactly. If it prints an older version (e.g., `3.29.0`), the npx cache is stale — run `npx clear-npx-cache` and retry.
3. **Web search:** `web_search --query "bitcoin price"`
   - ✅ Expected: Returns search results. If you get `FIRECRAWL_API_KEY not set`, the key is missing from the MCP client env config.
4. **Vault round-trip:** `vault_save` then `vault_search`
   - ✅ Expected: Save returns an entry ID. Search retrieves it. If this fails, Convex backend is unreachable — check `npx convex dev` is running or Convex deployment is active.
5. **Swap estimate:** `estimate_swap --fromToken USDC --toToken ETH --amount 50000000`
   - ✅ Expected: Returns a quote with `buyAmount`, `price`, `priceImpact`. If it fails, `ZX_API_KEY` or `ALCHEMY_API_KEY` is missing/invalid in Convex dashboard.
6. **No Buffer errors:** Start the MCP server
   - ✅ Expected: No `ReferenceError: Buffer is not defined`. If present, ensure Node.js >= 18 (`node --version`) and the MCP client spawns via `npx` (not a browser runtime).
7. **MCP client tools:** Restart MCP client after config change
   - ✅ Expected: Client shows all expected tools (100+). If 0 or partial tools appear, the client cached a stale connection — **quit and reopen** the client completely.
8. **Server-side monitors:** Check Trigger.dev connectivity (if configured)
   - ✅ Expected: `doctor` shows Trigger.dev as ✅. If `TRIGGER_SECRET_KEY` is unset, monitors/research shifts will fail — set it in the worker env.
9. **Chat responses:** Send a message to an agent
   - ✅ Expected: Full AI response, not "Running in local mode". If local mode appears, set an LLM API key (`GROQ_API_KEY`, `ANTHROPIC_API_KEY`, etc.) in the Convex dashboard.
10. **Convex deploy:** `npx convex dev`
    - ✅ Expected: Pushes without `Error:` lines. `[WARN]` messages are OK and can be ignored.
11. **No `internalAction` errors:** Deploy Convex functions
    - ✅ Expected: No `internalAction is not defined` errors. If present, add `internalAction` to the import from `./_generated/server`.
12. **No type errors:** Deploy Convex functions
    - ✅ Expected: No `Property 'X' does not exist on type` errors. If present, check `api.` → `internal.` migration missed callers with different argument shapes.
13. **No Vite template literal errors:** Start the dev server
    - ✅ Expected: No `Expected "}" but found "You"` errors. If present, a template literal is missing its closing backtick — check `systemPrompt: \`You are...` lines.
