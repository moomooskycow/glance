# Project: Glance

> **Disposition (2026-07-03): frozen, maintenance-only.** The GLANCE mission
> has been re-founded as [misty-step/glance](https://github.com/misty-step/glance),
> a greenfield Rust workspace, and that repo now owns the roadmap below —
> treat `Active Focus` and `Key Issues` here as historical, not a live plan.
> No new feature work lands in this repo; only fixes required to keep CI,
> security, and existing users unbroken. What changed architecturally: the
> successor generates a **sister repo** (`<name>-glance`, a durable,
> version-controlled mirror of generated HTML pages, updated via mirrored
> PRs) instead of writing `.glance.md` files inside the source tree — the
> in-repo generated-file pattern this project pioneered is exactly what the
> rewrite moves away from, following a real scar from generated files
> breaking a build elsewhere. This repo remains the reference implementation
> of that earlier pattern.

## Vision
CLI tool that recursively scans directories and generates high-quality `.glance.md` summaries using LLM failover chains.

**North Star:** The go-to developer tool for instant codebase comprehension — every directory self-documenting, always current, zero maintenance. Drop-in integration: one workflow file and your repo documents itself forever.

**Target User:** Developers who want quick, automated documentation of directory contents across codebases.

**Current Focus:** Two tracks in parallel:
1. **Reliability** — fix known bugs, clean up technical debt
2. **Zero-friction integration** — GitHub Actions workflow, post-merge hooks; make it trivial to adopt and keep fresh

**Key Differentiators:**
- Multi-provider LLM failover (Gemini, OpenRouter/Grok) for reliability
- Smart regeneration via modification time checking
- Respects .gitignore, handles large files gracefully
- Zero config for basic use, customizable prompts for advanced use

## Domain Glossary

| Term | Definition |
|------|-----------|
| `.glance.md` | Output file per directory containing LLM-generated summary |
| `glance.md` | Legacy read-only filename (read but not written) |
| `FallbackClient` | Composite LLM client that tries providers in order |
| Bottom-up processing | Leaf directories first; parents incorporate child summaries |
| Mod-time tracking | Skip regeneration if directory contents unchanged |
| Prompt template | Customizable instruction sent to LLM; default in `llm.DefaultTemplate()` |
| Tier | One LLM provider+model in the failover chain |

## Active Focus

- **Theme 1:** Reliability and correctness (fix sentinels, determinism, path issues, UTF-8 truncation)
- **Theme 2:** Zero-friction CI integration (composite action in `.github/actions/glance/`, auto-commit on by default)
- **Key Issues (Reliability / v1.1 / now):** #51, #53, #60, #66 — all `effort/s`. Note: all GitHub issues here were bulk-closed 2026-03-28 as part of a move to file-driven `backlog.d/` tracking, not because the work necessarily landed. Verified against code: #51 (relative paths, `glance.go:424`) and #53 (sorted keys, `sort.Strings` in `FormatFileContents`) are actually fixed. #60 (mutable sentinel errors, see `CLAUDE.md` Gotchas) and #66 (UTF-8 boundary truncation — `filesystem/reader.go:86` still does raw `content[:maxBytes]` byte-slicing with no rune-boundary check) are **not** fixed and have no `backlog.d/` entry; both remain open work despite showing CLOSED on GitHub.
- **Key Issues (CI Integration / Backlog / next):** #67 (action), #68 (docs)

## Quality Bar

- `go test -race ./...` — race detector mandatory
- `golangci-lint run --config=.golangci.yml` — zero warnings, never suppress
- `govulncheck ./...` — no known vulnerabilities
- No coverage gate yet (issue #32 tracks adding one)
- Conventional commits with detailed multiline messages

## Patterns to Follow

### Functional Options Config
```go
// Each With* returns new immutable copy
cfg := config.NewConfig().
    WithForce(true).
    WithPromptFile("custom.txt")
```

### Test Seam Injection
```go
// Function vars swapped in tests (6 seams in glance.go)
var generateSummary = service.Generate
// Tests: generateSummary = func(...) { return fakeResult, nil }
```

### Typed Errors
```go
return glanceerrors.New(glanceerrors.CodeLLMError, "provider unavailable").
    WithSuggestion("check GEMINI_API_KEY").
    WithCause(err)
```

## Lessons Learned

| Decision | Outcome | Lesson |
|----------|---------|--------|
| Three retry layers | Confusing, redundant | Only FallbackClient retries (fixed #64) |
| Absolute paths in LLM prompt | LLM gets machine-specific noise | Use relative paths (#51) |
| Go map for file contents | Non-deterministic prompt ordering | Sort keys or use ordered slice (#53) |

---
*Last updated: 2026-02-23*
*Updated during: /groom session*
