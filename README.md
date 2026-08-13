# Spec Pipeline

> PRD → FRD → Delivery Spec → Review → Export

A Claude Code plugin that transforms product requirements into structured delivery documents. Supports configurable role combinations: **F2E / Backend / DBA / App(Native)**, with QA consistency checks always enabled.

## What it does

Takes a PRD (Product Requirements Document) through 4 phases, with human checkpoints between each:

```
PRD ──→ [spec-extract] ──→ FRD ──→ [spec-deliver] ──→ Delivery Spec
                 ↑ human checkpoint         ↑ human checkpoint
                                                         │
         Export ←── [spec-export] ←── Review ←── [spec-review]
                                        ↑ human checkpoint
```

| Phase | Skill | What it does |
|-------|-------|-------------|
| 1 | `/spec-extract` | PRD → FRD: scans codebase for existing patterns, asks role-dependent structured questions one at a time |
| 2 | `/spec-deliver` | FRD → Delivery Spec (Markdown): system responsibility matrix + sequence diagrams + API contracts + network resilience strategy + DB schema |
| 3 | `/spec-review` | Multi-role fan-out review: selected roles + QA — subagents run in parallel, merged into Blocking/Ambiguous/Suggestion report |
| 4 | `/spec-export` | Format conversion: Markdown → HTML (self-contained, printable, Artifact preview) |

## Configurable Roles

At pipeline entry (Step 0), choose which roles are involved. This affects **every phase**:

| Key | Role | Scope |
|-----|------|-------|
| `f2e` | Frontend RD (F2E) | SPA/SSR routing, state management, API integration, rendering strategy, caching |
| `app` | App RD (Native) | Push integration, Deep Link, offline behavior, native components |
| `backend` | Backend RD | API design, data flow, scheduled jobs, concurrency safety, auth |
| `dba` | DBA | Schema design, target DB mapping, index strategy, read/write separation |
| `qa` | QA / Consistency | Cross-section checks, FRD↔Spec consistency, gap detection — **always enabled** |

### Presets

| Preset | Roles | Typical project |
|--------|-------|----------------|
| Web Full-stack | `f2e` + `backend` + `dba` | Admin systems, SaaS |
| App Full-stack | `app` + `backend` + `dba` | Native App features |
| Full Stack | `f2e` + `app` + `backend` + `dba` | Cross-platform features |
| Pure API | `backend` + `dba` | Microservices, API Gateway |
| Custom | User picks | Special requirements |

### How roles affect each phase

| Phase | Impact |
|-------|--------|
| spec-extract | Questions adjust by role (e.g., push/Deep Link only with `app`, SSR/CSR only with `f2e`) |
| spec-deliver | System matrix columns, sequence diagram participants, D-section scope, network resilience matrix |
| spec-review | Number and type of subagents (only selected roles + QA) |
| spec-export | No impact (format conversion is role-agnostic) |

## Install

```bash
# 1. Add as marketplace (one-time)
claude plugin marketplace add YenChiayou/spec-pipeline-for-RD

# 2. Install
claude plugin install spec-pipeline
```

## Usage

```bash
# Full pipeline with checkpoints
/spec-pipeline:spec-pipeline

# Start from a PRD file
/spec-pipeline:spec-pipeline path/to/prd.md

# Run individual phases (each is independently usable)
/spec-pipeline:spec-extract          # PRD → FRD
/spec-pipeline:spec-deliver          # FRD → Delivery Spec  
/spec-pipeline:spec-review           # Multi-role review
/spec-pipeline:spec-export           # Format conversion
```

## What the Delivery Spec covers

A single Markdown document (HackMD-compatible) for all selected RDs:

- **A. System Responsibility Matrix** — who builds what (by deliverable, not by story)
- **A2. Interact Pattern** — for chat-like UIs: renderMode routing, msgType matrix
- **B. Sequence Diagrams** — mermaid diagrams for every key flow
- **C. API Contracts** — C-0 through C-N, complete request/response/errors, per-API network error behavior
- **D. Platform Strategy** (role-dependent sections):
  - D-1. Auth Strategy — all roles
  - D-2. Push Payload — `app` only
  - D-2b. Real-time Communication — `f2e` only (if applicable)
  - D-3. Lifecycle — all roles
  - D-4. Post-lifecycle Behavior Matrix — all roles
  - D-5. **Network Resilience Strategy** — `app` or `f2e`
  - D-6. Routing Strategy — `app` or `f2e`
- **E. Database Schema** — full DDL (ANSI SQL) + target DB platform mapping + read/write separation + ER diagram (`dba`), or simplified version (`backend` only)

## Network Resilience (D-5)

When `app` or `f2e` roles are selected, the Delivery Spec includes an **API Network Behavior Matrix** — every API must answer 4 questions:

1. **Retry strategy** — auto retry count, interval (exponential backoff?), retryable status codes
2. **User perception** — what the user sees during loading → success → failure
3. **Degradation behavior** — can cached/local data fill in, or is it blocking?
4. **Optimistic vs pessimistic** — update UI before API confirms, or wait?

Plus role-specific sections:
- **App**: offline mode detection, available offline features, background request handling, token expiry + offline edge case
- **F2E**: SWR/staleTime config, Error Boundary granularity, SSR failure fallback, WebSocket reconnection

## Patterns baked in

| Pattern | Where |
|---------|-------|
| Cursor pagination (keyset, not offset) | spec-deliver C section |
| UNION ALL for mixed queries | spec-deliver C section |
| Atomic UPDATE for token validation | spec-deliver C + E sections |
| DB-native queue processing (e.g. `FOR UPDATE SKIP LOCKED` / `ROWLOCK + READPAST`) | spec-deliver E section |
| Sentinel value for UNIQUE INDEX with NULL (DB-aware) | spec-deliver E section |
| Post-lifecycle behavior matrix (every API) | spec-deliver D-4 |
| Per-API network error behavior matrix | spec-deliver D-5 |
| read-your-own-writes for read/write separation | spec-deliver E section |
| Project-type-aware question depth (consumer app vs admin vs SaaS vs API) | spec-extract |
| Multi-role parallel review with subagent fan-out | spec-review |
| Markdown → HTML structure mapping rules | spec-export |

## Design philosophy

- **Not fully automatic.** Each phase needs human confirmation — the most valuable decisions happen between phases, not inside them.
- **Each skill is independently usable.** `/spec-review` doesn't require running the full pipeline first.
- **Scans before asking.** Phase 1 reads your codebase (auth mechanism, push models, URL scheme, offline cache strategy) before asking questions, so questions are specific and grounded.
- **Review fans out, doesn't serialize.** Selected role subagents + QA run in parallel, each reviewing from their role's perspective.
- **Configurable, not hardcoded.** Role selection at entry shapes every downstream phase — no wasted chapters for roles not involved.

## Requirements

- Claude Code with Agent tool (for spec-review fan-out)
- Artifact tool (for preview publishing)

## License

MIT
