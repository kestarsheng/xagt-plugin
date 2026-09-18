# Verification evidence

## Prerequisites

- Review commit: `ed85265`
- API base URL: `https://code-review-agent-ashy-six.vercel.app/v1`
- Authentication: none

## 1. Health check

```bash
curl --fail --silent --show-error https://code-review-agent-ashy-six.vercel.app/health
```

Expected response:

```json
{"status":"ok","commit":"ed85265"}
```

## 2. Deployment proof

```bash
curl --fail --silent --show-error https://code-review-agent-ashy-six.vercel.app/.well-known/xagent-verification.json
```

Expected response:

```json
{"schemaVersion":1,"slug":"kestarsheng-code-review-agent","commit":"ed85265"}
```

## 3. Capability call

```bash
curl --fail --silent --show-error \
  --request POST https://code-review-agent-ashy-six.vercel.app/v1/review \
  --header "content-type: application/json" \
  --data '{"code":"def f(x):\n    return x / 0","language":"python"}'
```

Expected success response (abridged):

```json
{
  "ok": true,
  "language": "python",
  "model": "deepseek-chat",
  "report": {
    "summary": "...",
    "score": 30,
    "grade": "D",
    "issues": [
      {
        "severity": "critical",
        "category": "correctness",
        "line": 2,
        "title": "除零错误",
        "description": "...",
        "suggestion": "..."
      }
    ],
    "strengths": ["..."],
    "improvements": ["..."]
  }
}
```

## 4. Safe error behavior

Empty body:

```bash
curl --fail --silent --show-error \
  --request POST https://code-review-agent-ashy-six.vercel.app/v1/review \
  --header "content-type: application/json" \
  --data '{}'
```

Expected: HTTP 422 with `{"ok":false,"error":...}`.

Oversized code (> 60 000 chars): HTTP 413.
LLM provider failure: HTTP 502 with `{"ok":false,"error":"LLM 调用失败: ..."}`.

## 5. End-to-end PR review (real LLM call)

This section demonstrates the full pipeline: paste a GitHub commit URL → fetch diff → triple-engine review → structured report.

### Request

```bash
curl --fail --silent --show-error \
  --request POST https://code-review-agent-ashy-six.vercel.app/v1/review_pr \
  --header "content-type: application/json" \
  --data '{"url":"https://github.com/kestarsheng/code-review-agent/commit/952fa21","language":"python"}'
```

**Input:** A real commit from the project's own repository — the commit that added the `github_fetch.py` module (PR/commit URL diff fetching capability).

### Response (abridged)

```json
{
  "ok": true,
  "files_changed": [
    "app/github_fetch.py",
    "app/main.py",
    "app/mcp_server.py",
    "app/schemas.py",
    "tests/test_app.py"
  ],
  "added_lines": 240,
  "removed_lines": 0,
  "model": "deepseek-chat",
  "report": {
    "summary": "本次变更新增了从 GitHub PR/commit URL 拉取 unified diff 的能力…但存在一个 SSRF 风险点：fetch_diff 会跟随重定向且未限制最终主机…",
    "score": 46,
    "grade": "D",
    "dimension_scores": {
      "correctness": 36,
      "security": 9,
      "performance": 85,
      "maintainability": 88,
      "best_practice": 54
    },
    "issues": [
      {
        "severity": "critical",
        "category": "correctness",
        "line": 104,
        "title": "Python 语法错误，代码无法解析",
        "source": "rule",
        "rule_id": "PY-AST-S001",
        "confidence": 1.0
      },
      {
        "severity": "critical",
        "category": "security",
        "line": 229,
        "title": "使用 eval() 执行任意代码",
        "suggestion": "避免使用 eval()。如需解析表达式，使用 ast.literal_eval()。",
        "fix_code": "result = ast.literal_eval(x)",
        "source": "rule",
        "rule_id": "PY-S001",
        "confidence": 0.95
      },
      {
        "severity": "major",
        "category": "security",
        "line": 78,
        "title": "fetch_diff 跟随重定向且未校验最终主机，存在 SSRF 风险",
        "source": "llm",
        "confidence": 0.7
      },
      {
        "severity": "minor",
        "category": "correctness",
        "line": 31,
        "title": "PullRequestReviewRequest.url 缺少 max_length 限制",
        "source": "llm",
        "confidence": 0.7
      }
    ],
    "engine_info": {
      "rule_count": 8,
      "ast_count": 0,
      "llm_count": 4,
      "confirmed_count": 0,
      "total_rules_run": 8,
      "engines": ["rule", "ast", "llm"]
    }
  }
}
```

### What this proves

| Evidence | Detail |
| --- | --- |
| **GitHub URL → diff fetch** | `review_pr` endpoint accepted a commit URL, fetched the unified diff from GitHub API |
| **Triple-engine activation** | `rule_count=8` (rule engine fired 8 rules), `llm_count=4` (LLM found 4 issues), `engines=["rule","ast","llm"]` |
| **Cross-engine attribution** | Issues tagged with `source: "rule"` (PY-AST-S001, PY-S001, PY-B002) and `source: "llm"` (SSRF, max_length, newline, test coverage) |
| **Auto fix code** | `eval()` issue includes `fix_code: "result = ast.literal_eval(x)"` — directly replaceable |
| **5-dimension scoring** | correctness=36, security=9, performance=85, maintainability=88, best_practice=54 → composite score 46/D |
| **Change metadata** | `files_changed` (5 files), `added_lines` (240), `removed_lines` (0) extracted from diff |
| **12 issues total** | 2 critical + 1 major + 2 minor + 7 info — full severity spectrum exercised |
