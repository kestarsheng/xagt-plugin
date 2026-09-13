# Contract Guard

## Capability

- **One-line description:** Deterministic API breaking-change detector for AI agents — compares two OpenAPI / GraphQL / JSON Schema contracts and returns structured, reproducible findings with zero LLM dependency in the core diff.
- **Who it helps:** Any AI agent that needs to judge whether an API change is safe or breaking before merging, publishing, or upgrading a dependency. Also useful for developers and CI pipelines that want a deterministic compatibility gate.
- **Capability boundary:** Accepts two contract texts (old + new) as strings in OpenAPI 3.x (JSON/YAML), GraphQL SDL, or JSON Schema (draft-07 / 2020-12) format, plus a format identifier and optional `use_llm` flag. Max input 200 000 chars per spec. Returns a JSON report with `breaking` (bool), `breaking_count`, `total_changes`, per-severity counts, and a `findings[]` array where each finding has `change_type`, `breaking`, `severity` (critical/major/minor/info), `location`, `summary`, `source` ("confirmed" for deterministic engine, "advisory" for optional LLM layer), and `id`. The deterministic engine covers 35+ change types across three formats including content-type changes, deprecation detection, GraphQL directive changes, and JSON Schema 2020-12 keywords (prefixItems, contains, dependentRequired, unevaluatedProperties). Also provides 6 MCP tools (`check_breaking_changes`, `list_supported_formats`, `explain_change_type`, `suggest_version_bump`, `generate_changelog`, `suggest_migration`), 6 REST endpoints (`POST /v1/diff`, `POST /v1/chain-diff`, `POST /v1/semver`, `POST /v1/migration`, `POST /v1/sarif`, `POST /v1/changelog`), SARIF 2.1.0 export for GitHub Code Scanning, and an interactive demo page. Does not execute code, call external APIs (unless LLM advisory enabled), or persist submitted contracts.

## Live API

- **API base URL:** https://contract-guard-eta.vercel.app/v1
- **Health-check URL:** https://contract-guard-eta.vercel.app/health
- **Authentication:** none
- **Rate limits / known limits:** Max spec size 200 000 chars per request. Vercel serverless 10 s timeout — sufficient for all deterministic diffs. LLM advisory layer (optional) may add latency.
- **API contract:** OpenAPI at `/docs`; request `POST /v1/diff` body `{"format": "openapi|graphql|json-schema", "old_spec": string, "new_spec": string, "use_llm"?: bool}`, response `{"schema_version": 1, "format": string, "breaking": bool, "total_changes": int, "breaking_count": int, "counts": {...}, "summary": string, "llm_enabled": bool, "findings": [...]}`. Additional endpoints: `POST /v1/chain-diff` (multi-version analysis), `POST /v1/semver` (SemVer bump), `POST /v1/migration` (migration suggestions), `POST /v1/sarif` (SARIF export), `POST /v1/changelog` (markdown changelog). MCP endpoint at `/mcp` with 6 tools. Demo page at `GET /`.

## Source and reproducibility

- **Source repository:** https://github.com/kestarsheng/contract-guard
- **Review commit:** `e45354dc626edcc04657d951b55ea4442058c3bb`
- **Source submitted in this PR:** `source/`
- **Run tests:** `pip install -r requirements.txt && pytest tests/ -v`
- **Run locally:** `pip install -r requirements.txt && uvicorn app.main:app --reload`
- **Deploy:** Vercel (current production deployment). Set `COMMIT` env var to the git commit hash.
- **Version binding:** `GET /health` returns `{"status":"ok","commit":"<commit>","service":"contract-guard","version":"1.0.0"}`; `GET /.well-known/xagent-verification.json` returns `{"schemaVersion":1,"slug":"kestarsheng-contract-guard","commit":"<commit>"}`. The commit is injected via the `COMMIT` environment variable at deploy time.

The API must expose:

```json
// GET /health
{"status":"ok","commit":"e45354dc626edcc04657d951b55ea4442058c3bb","service":"contract-guard","version":"1.0.0"}
```

```json
// GET /.well-known/xagent-verification.json
{"schemaVersion":1,"slug":"kestarsheng-contract-guard","commit":"e45354dc626edcc04657d951b55ea4442058c3bb"}
```

## Verification

The reproducible call instructions and example responses are in `verification/README.md`.

- **Health-check result:** `{"status":"ok","commit":"e45354dc626edcc04657d951b55ea4442058c3bb","service":"contract-guard","version":"1.0.0"}`
- **Capability call:** `POST /v1/diff` with `{"format":"openapi","old_spec":"...","new_spec":"..."}`
- **Expected error behavior:** Invalid format → 400; oversized spec → 413; malformed JSON → 422.

## Security and data handling

- **Data collected:** Two contract texts (old + new) and a format identifier. No authentication, no user identifiers.
- **Purpose and retention:** Contracts are parsed in memory for diff computation only. The service does not persist submitted contracts to any database or log. If LLM advisory is enabled, contract excerpts may be sent to the configured LLM provider.
- **Third parties / outbound network calls:** None in default mode (deterministic engine only). Optional LLM advisory calls an OpenAI-compatible API (DeepSeek) if `LLM_API_KEY` is configured.
- **Secrets:** No secrets are committed. `LLM_API_KEY` is set as a deployment environment variable and never appears in source.
- **Known risks / restrictions:** Vercel serverless functions have a 10 s timeout; the deterministic engine completes well within this limit. The optional LLM advisory layer may timeout on very large specs.

## Support

- **Team / builder:** kestarsheng (刘宇珂)
- **Contact:** 2410251355@henu.edu.cn
- **License / rights:** MIT — submission for X-Agent AI MCP Hackathon 2026. Submitter owns all source and authorizes review and post-award retention.