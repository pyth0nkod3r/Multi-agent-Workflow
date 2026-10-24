# Git & GitHub Adoptions — Standing Doctrine v1.0

How version control is done in this harness: repository conventions, the
GIT-HYGIENE write protocol, commit-marker enforcement, and the parallel-session
playbook. Born from real incidents — every rule here has a motivating case.

**Scope:** all repos, all sessions, current and future. Multiple agent sessions
share ONE working tree and ONE local repo per project — every rule below
follows from that fact.

---

## 1. Repository Conventions

- **Per-project repos** under the `pyth0nkod3r` GitHub account; **private by
  default** (`gh repo create <name> --private`).
- **`gh` CLI is authenticated** as `pyth0nkod3r` — used for repo create/rename;
  SSH push from the workspace key (occasional transient push failures → just
  retry, do not debug the first failure).
- **Rename protocol:** `gh repo rename --yes` (GitHub auto-redirects old URLs).
  NEVER `mv` a project directory inside the workspace rootfs — the file-sync
  layer EPERM-locks `.git/objects` (`bad object HEAD`, `git fsck` EPERM) and it
  does **not** self-heal. Correct sequence: commit + push + verify remote
  **before** any rename, then **fresh clone** instead of `mv`.
  *(Motivating incident: EquityScout rebrand, 2026-10-14.)*
- **Git identity (user-set, global):** Miracle Anyanwu /
  `o.amagwula@stu.unizik.edu.ng`; `pyth0nkod3r@users.noreply.github.com` kept as
  local overrides in the platform + multiagent repos.
- **`AGENTS.md` convention** — repo agent-docs live in `AGENTS.md`, NOT
  `CLAUDE.md` (Codex/Kilo/Antigravity all read AGENTS.md).
- **Path rule:** never rewrite paths in repo files — repo files stay
  byte-identical across harnesses; desktop agents map artifacts onto their own
  mechanisms, never fabricate harness-specific syntax.

## 2. GIT-HYGIENE Write Protocol (mandatory, every commit)

Motivating incident `6ac1e91`: a parallel session re-applied a reverted fix,
the stale commit deployed, backend down ~12h. The protocol exists so that
cannot recur:

1. **NEVER blind `git pull`.** Pull is unnecessary for same-file races
   (protection = disjoint file ownership + verify-before-fix) and commit races
   (shared local repo, index-lock serialized; any push carries the whole local
   chain). It is **dangerous**: a merge/checkout on a dirty tree is the
   destructive-op incident class.
2. **Pre-commit sequence, every time:**
   `git fetch` → `git log HEAD..origin/main` drift check → **re-read touched
   files** (verify-before-fix) → commit → push → **verify remote HEAD moved**.
3. **Pull is sanctioned ONLY if** remote moved AND the working tree is clean.
4. **Orchestrator-only commits** — builders never `git commit`/`push`
   (enforced in the builder roster profile).
5. **Run-id markers + orphan-commit audit** — every commit traceable to a run.

## 3. Commit Conventions & Enforcement

- Subject format: `type(scope): description [run <runid>]` — the `[run <id>]`
  marker is **mandatory**.
- **Enforced by a wired `commit-msg` hook**: `.githooks/commit-msg`, activated
  per-clone via `git config core.hooksPath .githooks`. Hook-tested reject/allow.
  The hook **rejects** non-conforming markers — e.g. a `[hotfix]` marker was
  bounced 2026-10-23; the fix was re-marked with a run id.

## 4. Parallel-Session Playbook

Multiple sessions work one tree. Rules of engagement:

- **Scoped `git add` only** — name your files explicitly, **never `-A`** over a
  shared tree (in-flight foreign work must not be swept into your commit).
- **Absorbing parallel commits** (remote moved, tree clean): `git pull
  --ff-only` + `stash/pop` for local WIP, preserving in-flight work
  byte-identical. *(Verified pattern, 2026-10-23 remote-sync: absorbed 4
  parallel commits / 23 files without losing a WIP rebuild plan.)*
- **Same-file races:** protection = disjoint file ownership by unit +
  verify-before-fix re-read of touched files immediately before commit.
- **Never overlap long QA runs** (e.g. pytest) across sessions.

## 5. Destructive-Op & Environment Rules

- **No blind pull** (§2), **no `mv` of project dirs** (§1).
- **Exit-code discipline:** never `cmd | tail; echo $?` or `cmd | head; echo $?`
  — that captures tail/head's exit code, not the command's. Redirect to a file,
  then `echo $?` (or use `PIPESTATUS`). *(Bug class hit twice — once in prod
  with a no-op tsc gate.)*
- `UV_LINK_MODE=copy` in synced trees.
- Post-cancel restore comes from `git HEAD`, never from ad-hoc fixes.
- Frontend typecheck gate: `tsc -b`, never root `tsc --noEmit` (the root
  tsconfig has `files: []` + project references — a no-op gate).

## 6. Doctrine Repos & Sync Flow

- **Multi-agent-Workflow** (this repo) = canonical doctrine: roster/,
  ORCHESTRATION.md, knowledge/. Edits land here first.
- **ide-setup** = the IDE deployment package; sync flow is
  **canonical → package, canonical wins on conflict**. Never blind-overwrite
  deliberate IDE adaptations (e.g. the inline `rikkahub-orchestration` skill).
- Research/product docs live in each project's own repo
  (`docs/RESEARCH-*.md`, `tasks/<runid>/`, `runs/`).

## 7. Open Items

- `ai-job-search` fork merge pending (134 commits ahead, 8 conflict files).
- Global git identity vs local noreply overrides: user may want the noreply
  identity synced global-side.

## Cross-References

- `knowledge/security-adoptions.md` — the security doctrine companion
- `knowledge/code-quality.md` — quality gates referenced by the QA exit rules
- `../ORCHESTRATION.md` — git-commit-hygiene block (canonical protocol text)
- `../roster/builder.md` — GIT HYGIENE section (builder-side enforcement)

## Changelog

- 2026-09-07 — orchestrator-only commits + run-id markers (LRN-20260907-005)
- 2026-09-14 — amendment: commit-marker rules extended
- 2026-09-29 — full GIT-HYGIENE set after incident `6ac1e91`; commit-msg hook wired
- 2026-10-14 — `mv`-directory EPERM lesson codified (EquityScout rebrand)
- 2026-10-23 — `[hotfix]` rejection proves hook enforcement; ff-only+stash absorb pattern verified
- 2026-10-24 — this doctrine doc committed [run 20261024-1450-git-github-adoptions]
