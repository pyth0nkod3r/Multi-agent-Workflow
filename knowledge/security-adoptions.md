# Security Adoptions — Standing Doctrine v1.0

The harness-wide security posture: what is adopted, where it is enforced, and
what every agent (orchestrator and sub-agents alike) must do. This doc is the
summary; the full texts live in the cross-referenced files.

**Scope:** all projects current and future. **Adopted by the user** — these are
directives, not suggestions.

---

## 1. Gate 8 — Zero-Day Audit (user-set 2026-09-22)

Every ruff/lint error must be audited as a potential zero-day **before any
suppression**.

- **No blind `noqa`** and no per-file-ignore for anything that could be a real
  bug, exploitable loophole, or security vulnerability.
- Suppression is acceptable **only** for verified false positives (e.g.
  parameterized SQL flagged as injection, list-form subprocess flagged as shell
  injection) **with an inline comment explaining WHY it is safe**.
- Real issues are **fixed, not silenced**.
- Applies to the orchestrator **and** every sub-agent (builder, critic,
  planner) during their QA runs.
- Every dispatch brief for substantive code work must carry the line:
  *"During QA, audit every ruff error as a potential zero-day — no blind noqa
  for real vulnerabilities; fix or verify-safe-with-comment."*

Full text: `knowledge/code-quality.md` (gate 8).

## 2. Gate 9 — SEC (shipped 2026-09-28)

- The **critic is the security agent** (deliberate decision — no separate
  profile). Checklist-driven QA is its native mode; a separate agent is used
  only for live offensive engagements (rare, user-supervised, one-off dispatch
  against `security-resources.md`).
- The critic runs the relevant security checklist on **every substantive diff**.
- CI security findings follow the gate-8 rule (audit before suppress).
- **Evidence-hygiene redaction rule:** secrets, tokens, and credentials never
  appear in logs, reports, run ledgers, or committed docs.

Full text: `knowledge/code-quality.md` (gate 9).

## 3. Security Knowledge Base

| Asset | What it holds |
|---|---|
| `knowledge/security-resources.md` | 30.8KB digest: lostsec article (967-line pdftotext), BugHunter 7 skills + OSINT 4 read; **15 live-verified resources** (OWASP Top 10 2025, ASVS 5.0, NIST SSDF, WSTG, STRIDE, Threat Dragon, Vault, trivy, gitleaks, semgrep — with its LGPL-2.1 licensing flag noted; SLSA explicitly UNVERIFIED) |
| `knowledge/security-checklists/` | 5 defensive checklists distilled from BugHunter (MIT + CC-BY): `api-security` (56L), `auth-security` (45L), `business-logic` (43L), `code-review` (39L), `mobile-apk-release` (41L — the APK release gate for Android projects) |

All checklist files carry **DEFENSIVE USE ONLY** headers.

## 4. Offensive-Material Rule (user-set)

Offensive skills/materials are used **only** as defensive checklists against
**our own products**. Never as attack tooling against third parties.

## 5. CI Security (reference instance: platform repo, commit `9b84c9d`)

- `.github/workflows/security.yml`
  - **gitleaks — blocking** (`@v2`, `fetch-depth: 0` so full history is scanned)
  - **trivy** — filesystem scan, advisories HIGH + CRITICAL, `continue-on-error`
- `.github/dependabot.yml` — weekly: pip (`/backend`), npm (`/frontend`), github-actions
- `docs/security/README.md` — documents the zero-day audit rule for contributors

New repos should replicate this trio; copy the reference files rather than
reinventing them.

## 6. Secrets & Credential Discipline

- **Env-only by design:** DB/config rows store `*_redacted` markers only; raw
  keys live exclusively in environment variables on the host (e.g. Render env
  vars for provider keys, cron keys, LLM failover chain).
- Device-side secrets live in `/workspace/.secrets/` (chmod 600), one file per
  credential; never committed.
- **Cron-key rotation after ANY DB-reset incident:** a Neon DB reset wipes
  `auth_tokens` — after any DB incident, check `/admin/api-keys` and rotate the
  cron Bearer key (lesson from the 28–29 Sept reset).
- MCP headers are redacted in display layers but stored verbatim in config —
  treat MCP configs as secrets-bearing.

## 7. Agent-Surface Hardening (harness level)

- **MCP "Always Allow" intentionally not offered** for `mcp_add`/`mcp_update` —
  every install/change is per-call confirmed (a hostile MCP server can
  exfiltrate everything the assistant can reach).
- `browser_eval_js` runs HARDLINE checks pre-execution: shell-shaped strings,
  `document.cookie` writes, `eval`/`Function` constructors, and string-form
  `setTimeout` are all blocked.
- Path-safety guard on all file/batch tools; zip extraction rejects `../`
  traversal entries.
- External Automation Intent API defaults **OFF**; only explicitly trusted
  caller packages may fire intents.
- SSH host-key changes require explicit user confirmation before
  `ssh_forget_host_key` (MITM caution); saved hosts keep credentials out of
  chat history.
- **Known risk on record:** dispatched sub-agents on the native path receive the
  **full tool catalog**. Mitigation: only task-following models on sub-agent
  profiles (see the model-suitability scorecard — loose models leak identity and
  ignore constraints).

## 8. Cross-References

- `knowledge/code-quality.md` — gates v4 + gate 8 + gate 9 full texts
- `knowledge/security-resources.md` — the verified resource registry
- `knowledge/security-checklists/` — the per-area critic checklists
- Platform repo: `docs/security/README.md`, `.github/workflows/security.yml`,
  `.github/dependabot.yml`

## Changelog

- 2026-09-22 — gate 8 zero-day audit adopted (user directive)
- 2026-09-27 — security-resources + 5 checklists researched and committed (`7d189de`, `91f013c`)
- 2026-09-28 — SEC CI trio + gate 9 shipped to the platform repo (`9b84c9d`, `52f2f1a`); critic designated security agent
- 2026-10-24 — this doctrine doc committed [run 20261024-1435-security-adoptions]
