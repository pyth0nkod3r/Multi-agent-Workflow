# RESEARCH: Production TeX strategy for Render free tier (SPIKE)

> Run 20260917-p6-wave1 · research-only · sources: render.com docs/pricing
> (2026), tectonic-typesetting issues #1165/#319, TeX.SE minimal-install
> threads, justinmckelvey.com Render free-tier audit (Sept 2026). Written
> inline by orchestrator 17 Sept 2026 after 2 researcher fast-fail deaths.

## Constraint base (verified)
- Render free web service: 512 MB RAM, fractional CPU, 15-min spin-down,
  ephemeral FS (no persistent disk on free), 750 hrs/mo shared.
- Render build pipeline: separate 2 CPU/8 GB compute, Hobby = 500 build
  minutes/mo, disk cap 16 GB, 120-min build timeout. Build installs do NOT
  persist into the runtime image limits but ARE re-run every deploy (ephemeral).
- No Docker (user directive). Backend = Python/FastAPI, uv.
- latex.py already degrades gracefully (real: False) when engines missing.

## Options compared

| Option | Size | Build-time cost | Fidelity | PII risk | Verdict |
|---|---|---|---|---|---|
| texlive-full via apt | 5–8 GB | 15–30 min/deploy, eats 500 min/mo | full | none | ✗ absurd on free tier |
| TinyTeX / TL scheme-basic (apt or TUG) | 150–265 MB | ~5–10 min/deploy | good (XeTeX incl.) | none | viable fallback |
| **Tectonic** (single static binary, ~12–20 MB + ~40–50 MB frozen bundle cache, baked at build via curl) | ~60 MB | ~1 min | XeTeX-class; moderncv-compatible | bundle download is fixed CTAN content — no user PII leaves | **PRIMARY** |
| Precompiled template PDFs + pypdf overlay | ~0 | ~0 | poor (breaks moderncv layout logic; CV sections reflow) | none | ✗ kills the "tailored CV" quality bar |
| Client-side wasm TeX | 10–100 MB download/user | none server-side | XeTeX subset, slow first load | none | deferred — v2+ |
| External compile APIs (e.g. latex.ytotech, LaTeX.cc) | 0 | 0 | full | **HIGH — CV PII to third party** | ✗ violates privacy principle §2.3 |

## Known Tectonic caveats (checked, non-blocking)
- In-process memory leak (#1165, tectonic 0.15 library use) — NOT applicable:
  EaseApply shells out (`subprocess`) per compile, like the current latex.py
  engine pattern; process teardown clears it.
- Memory bounds on huge docs (#319) — CVs/cover letters are trivial documents;
  no pgfplots. Non-issue.
- On-demand package downloads at first compile → bake a frozen bundle/cache in
  the build step (`curl` binary + pre-compile canary once during build) so the
  runtime never needs CTAN and cold-start stays <2 s.
- glibc: Tectonic's GNU release targets x86_64/aarch64 glibc Linux — matches
  Render's Ubuntu-native (non-Docker) runtime.

## DECISION
- **Primary: Tectonic**, vendored at build time (download static binary +
  pre-warm/freeze bundle cache, ~60 MB total), invoked via the SAME
  subprocess path latex.py already uses — add `tectonic` as a third engine in
  `engine_available()`/`compile_tex` probing. Zero change to callers.
- **Fallback: TinyTeX/scheme-basic** (265 MB, ~2:40 install) if Tectonic hits
  a package-compat wall with moderncv (it is XeTeX-derived, moderncv works).
- Ship order: gate behind env var `TEX_ENGINE=tectonic|texlive|none` (default
  probe order texlive→tectonic→none) so dev keeps TeX Live untouched.
