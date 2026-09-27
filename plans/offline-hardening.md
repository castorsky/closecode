# Offline Hardening Plan — opencode fork for air-gapped use

## Goal

Ensure this tool **never initiates a network connection** except to:
- local LLM endpoints (Ollama, llama.cpp, LM Studio, any user-configured `openai-compatible` URL),
- user-explicitly-configured remote MCP servers (treated like a configured endpoint).

Strategy: **minimal, reviewable patches** — one small change per egress point, each with a short comment explaining why. No new abstractions, no config system changes beyond forcing existing flags.

## Egress inventory & patch list

### 1. Agent web tools (category 1)

| # | File | Change |
|---|------|--------|
| P1 | [`packages/opencode/src/tool/registry.ts`](packages/opencode/src/tool/registry.ts:209) | Remove `fetch` and `search` from the `Effect.all({...})` tool map (lines 218, 220) and from the `builtin` array (lines 241, 243). Comment: `// OFFLINE: webfetch/websearch removed — no outbound HTTP allowed`. The LLM will then never see these tools. |

Files [`tool/webfetch.ts`](packages/opencode/src/tool/webfetch.ts) and [`tool/websearch.ts`](packages/opencode/src/tool/websearch.ts) stay untouched (dead code, easy to re-enable).

### 2. Auto/background calls (category 2)

| # | File | Change |
|---|------|--------|
| P2 | [`packages/core/src/models-dev.ts`](packages/core/src/models-dev.ts:160) | In `populate` (~line 217), force the existing offline path: replace `if (Flag.OPENCODE_DISABLE_MODELS_FETCH) return {}` with an unconditional early `return {}` after disk/snapshot loads, commented `// OFFLINE: never fetch models.dev catalog; rely on OPENCODE_MODELS_PATH / local cache`. Model resolution then works from a locally-provided `OPENCODE_MODELS_PATH` JSON or the user's config-defined providers. |
| P3 | [`packages/opencode/src/effect/runtime-flags.ts`](packages/opencode/src/effect/runtime-flags.ts:21) | Force existing flags on: `disableExternalSkills: true` and `disableLspDownload: true` (hardcode to `true` with an `// OFFLINE:` comment). This kills, in one line each: remote skill discovery ([`skill/discovery.ts`](packages/core/src/skill/discovery.ts)) and **all ~10 LSP binary downloads** from GitHub/eclipse.org/JetBrains/HashiCorp in [`lsp/server.ts`](packages/opencode/src/lsp/server.ts) (each already early-returns on `flags.disableLspDownload`). |
| P4 | [`packages/core/src/ripgrep/binary.ts`](packages/core/src/ripgrep/binary.ts:104) | Before the download block (~line 104), throw a clear error instead of fetching from GitHub releases: `throw new Error("OFFLINE build: ripgrep not found locally — install 'rg' on PATH or pre-place it in " + Global.Path.bin)`. Single choke point; system-`rg` and cached-binary paths above still work. |
| P5 | [`packages/opencode/src/installation/index.ts`](packages/opencode/src/installation/index.ts:146) | In `info()` (~line 170): return `{ version: InstallationVersion, latest: InstallationVersion }` without calling `latest()`. Comment `// OFFLINE: no update checks`. This removes the npm/brew/choco/scoop/GitHub release-API calls (lines 147–259) from every startup path. The TUI "new version" banner disappears; `opencode upgrade` still exists but is unreachable via UI since latest == current. |

### 3. Cloud provider plugins with OAuth / hosted APIs (category 3)

| # | File | Change |
|---|------|--------|
| P6 | [`packages/opencode/src/plugin/index.ts`](packages/opencode/src/plugin/index.ts:67) | In `internalPlugins()`, comment out the entries that perform OAuth device-flow / hosted-API calls: `CodexAuthPlugin` (OpenAI/ChatGPT), `CopilotAuthPlugin` (GitHub Copilot), `ModalPlugin`, `GitlabAuthPlugin`, `PoeAuthPlugin`, `CloudflareWorkersAuthPlugin`, `CloudflareAIGatewayAuthPlugin`, `AzureAuthPlugin`, `DigitalOceanAuthPlugin`, `SnowflakeCortexAuthPlugin`, `XaiAuthPlugin`, `CerebrasPlugin`. Keep the array (possibly empty) with a single comment: `// OFFLINE: cloud/OAuth provider plugins removed — only local LLM endpoints are allowed`. |

Note: plain API-key providers (Anthropic, OpenAI-compatible, etc.) remain usable because they only connect to whatever URL the user configures — pointing them at Ollama/LM Studio works unchanged. The `opencode` hosted provider ([`core/src/plugin/provider/opencode.ts`](packages/core/src/plugin/provider/opencode.ts)) is unreachable without an opencode.ai account; optionally also comment its registration if found in the core plugin list (verify during implementation).

### 4. GitHub Actions integration (category 5)

| # | File | Change |
|---|------|--------|
| P7 | [`packages/opencode/src/cli/cmd/`](packages/opencode/src/cli/cmd/) — wherever `github.handler.ts` is registered in the CLI command tree | Comment out the registration of the `opencode github` subcommand (the handler file itself stays). This removes all `api.opencode.ai` / `api.github.com` calls ([`github.handler.ts:325`](packages/opencode/src/cli/cmd/github.handler.ts#L325), [`:998-1006`](packages/opencode/src/cli/cmd/github.handler.ts#L998), [`:1596`](packages/opencode/src/cli/cmd/github.handler.ts#L1596)). |

### 5. Web UI upstream proxy (category 6)

| # | File | Change |
|---|------|--------|
| P8 | [`packages/opencode/src/server/shared/ui.ts`](packages/opencode/src/server/shared/ui.ts:78) | In `serveUIEffect`, replace the fallback that proxies to `https://app.opencode.ai` (lines 88–106) with a local response: `return HttpServerResponse.text("Web UI unavailable in offline build", { status: 503 })`. Comment `// OFFLINE: do not proxy to app.opencode.ai`. |
| P9 | [`packages/opencode/src/server/shared/ui.ts`](packages/opencode/src/server/shared/ui.ts:12) | Tighten the CSP `connect-src *` (line 12) so a browser tab cannot phone home directly: restrict to `'self' http://localhost:* http://127.0.0.1:*` plus any user-configured MCP/LLM hosts if trivially derivable; otherwise just `'self' localhost 127.0.0.1`. |

### 6. Session sharing (found during implementation)

| # | File | Change |
|---|------|--------|
| P10 | [`packages/opencode/src/share/share-next.ts`](packages/opencode/src/share/share-next.ts:23) | Force `disabled = true` — the share service uploads **full session content (messages + code diffs)** to `opncd.ai`, which would leak enterprise code. Not in the original 9-patch list; added during implementation. |

### Explicitly NOT patched (allowed by design)

- **Remote MCP servers** — user config, same trust level as a local LLM URL ([`mcp/index.ts`](packages/opencode/src/mcp/index.ts)).
- **Local server / TUI / desktop IPC** — `http://localhost:4096`, WebSocket PTY, worker RPC.
- **Provider requests to user-configured URLs** (Ollama etc.) via the AI SDK in [`provider/provider.ts`](packages/opencode/src/provider/provider.ts).
- **OpenTelemetry OTLP exporter** ([`core/src/observability/otlp.ts`](packages/core/src/observability/otlp.ts)) — only activates if a user sets an endpoint; leave as-is (documented opt-in).
- **`opencode import <url>`** ([`cli/cmd/import.ts:134`](packages/opencode/src/cli/cmd/import.ts#L134)) — explicit user action fetching a share URL the user typed; same trust level as pasting content.
- **Wellknown remote config** ([`config/config.ts:376`](packages/opencode/src/config/config.ts#L376)) — only fetched for `type: "wellknown"` auth entries the user explicitly configured with a URL (e.g., an internal SSO endpoint).

## Verification steps

1. `bun typecheck` from `packages/opencode` and `packages/core`.
2. Grep audit: `grep -rn "fetch(\|HttpClientRequest.get\|new WebSocket" packages/*/src` — every remaining hit must be either local-only, user-config-driven (MCP/provider URL), or behind a documented opt-in flag.
3. Runtime smoke test with Ollama configured as the only provider: start session, run a prompt; confirm no outbound connections via `lsof -i` / proxy logging while idle and during a turn.
4. Confirm `webfetch`/`websearch` are absent from the tool list exposed to the model (`opencode debug v2` or TUI tools view).

## Patch count summary

10 patches (P1–P10), each ≤ ~15 lines, all marked with `// OFFLINE:` comments — grep for `OFFLINE` to review or selectively re-enable any of them.
