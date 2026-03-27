# breadcrumb — Context Compression Skill for Claude Code

> **Load full project understanding in ~300 tokens instead of ~40,000.**

A Claude Code skill that replaces reading the raw codebase at session start with a structured, always-fresh snapshot of your project's architecture, decisions, and pending work.

---

## The Problem

Every new Claude Code session, the AI re-reads your codebase from scratch.

In a real project, that costs **40,000–60,000 tokens before writing a single line of code**. 95% of those reads are redundant — the code didn't change.

The root cause: Claude doesn't have a "memory" of what your project is. It only has what it reads in the current context window.

**breadcrumb** solves this without requiring any special infrastructure — just two markdown files and a skill.

---

## How It Works

The system has three components:

| Component | Location | Purpose |
|-----------|----------|---------|
| `system-map.md` | `.claude/context/system-map.md` | Compressed architecture map (~400–600 tokens) |
| `session-log.md` | `.claude/context/session-log.md` | Last 5 sessions: what changed and **why** |
| `breadcrumb.md` | `~/.claude/skills/breadcrumb.md` | Instructions that tell Claude how to use the above |

Instead of reading 50 files, Claude reads 2 files and a git diff. It understands the same codebase with 98% fewer tokens.

---

## Technical Flow

### Session Start — `/breadcrumb load`

```
USER RUNS: /breadcrumb
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│  Claude reads .claude/context/system-map.md             │
│  (~400–600 tokens)                                      │
│  → Full architecture understanding                      │
│  → File index with one-line descriptions                │
│  → Active features and their status                     │
│  → Known bugs and non-obvious decisions                 │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Claude reads .claude/context/session-log.md            │
│  (~100–200 tokens)                                      │
│  → What happened in last session                        │
│  → What is pending right now                            │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Compare system-map git hash vs. current HEAD           │
│  git diff <stored_hash> HEAD --name-only                │
│                                                         │
│  No changes? ─────────────────────────────────────────▶ DONE
│                                                         │
│  Files changed?                                         │
│    └─▶ Read ONLY the changed files                      │
│         (skip: lockfiles, assets, auto-generated)       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  ✅ Context loaded. Claude reports:                     │
│                                                         │
│  📁 Project: MyApp                                      │
│  🔧 Stack: Next.js 14 | Supabase | Vercel | Tailwind    │
│  📋 Last session: 2026-03-25 — SEO audit                │
│  🔄 Changed files: 2 → app/layout.tsx, lib/i18n.ts      │
│  📌 Pending: fix /contact, create llms.txt              │
└─────────────────────────────────────────────────────────┘

Total tokens used: 300–800 (vs. 40,000+ without breadcrumb)
```

---

### Session End — `/breadcrumb update`

```
USER RUNS: /breadcrumb update
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│  Claude identifies session changes:                     │
│  • Files touched this session                           │
│  • Decisions made (and WHY)                             │
│  • Tasks completed                                      │
│  • What remains pending                                 │
└────────────────────┬────────────────────────────────────┘
                     │
           ┌─────────┴──────────┐
           ▼                    ▼
┌──────────────────┐  ┌──────────────────────────────────┐
│  system-map.md   │  │  session-log.md                  │
│                  │  │                                  │
│  Edit only the   │  │  Prepend new entry:              │
│  lines that      │  │                                  │
│  changed:        │  │  ## 2026-03-27 — [title]         │
│  • FILE INDEX    │  │  Done: [tasks]                   │
│  • FEATURES      │  │  Files: [list]                   │
│  • BUGS          │  │  Decisions: [WHY]  ← most value  │
│  • PENDING       │  │  Next: [steps]                   │
│  • header hash   │  │                                  │
│                  │  │  Keep only last 5 entries        │
│  Max 800 tokens  │  │                                  │
└──────────────────┘  └──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│  ✅ Breadcrumb updated                                  │
│  📝 system-map.md — 3 lines changed                     │
│  📓 session-log.md — new entry added                    │
└─────────────────────────────────────────────────────────┘

Cost: ~200 tokens
```

---

### First-Time Setup — `/breadcrumb init`

```
USER RUNS: /breadcrumb init  (only once per project)
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│  Explore codebase structure                             │
│  • ls, find (filtered), package.json / go.mod           │
│  • git log, current HEAD hash                           │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Read key architectural files selectively               │
│  ✅ Read: entry points, config, auth, types, schema     │
│  ❌ Skip: tests, lockfiles, assets, generated files     │
└────────────────────┬────────────────────────────────────┘
                     │
           ┌─────────┴──────────┐
           ▼                    ▼
┌──────────────────────┐  ┌──────────────────────────────┐
│  Generate            │  │  Generate                    │
│  system-map.md       │  │  session-log.md              │
│                      │  │                              │
│  STACK               │  │  Initial entry with          │
│  ARCHITECTURE        │  │  current project state       │
│  FILE INDEX          │  │  and what was explored       │
│  ACTIVE FEATURES     │  │                              │
│  KNOWN BUGS          │  │                              │
│  PENDING             │  │                              │
└──────────┬───────────┘  └──────────────┬───────────────┘
           │                             │
           └─────────────┬───────────────┘
                         ▼
              Write to .claude/context/
              (create directory if needed)
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  ✅ Breadcrumb initialized                              │
│  📁 .claude/context/system-map.md                       │
│  📓 .claude/context/session-log.md                      │
│                                                         │
│  From now on:                                           │
│    /breadcrumb        → session start                   │
│    /breadcrumb update → session end                     │
└─────────────────────────────────────────────────────────┘
```

---

## Token Savings

| Scenario | Without breadcrumb | With breadcrumb | Savings |
|----------|--------------------|-----------------|---------|
| Typical session (0–2 files changed) | ~40,000 | 300–800 | **98%** |
| New feature (10 files changed) | ~40,000 | 5,000–12,000 | **75%** |
| Questions / consulting | ~40,000 | 300 | **99%** |
| Massive refactor (everything changed) | ~40,000 | ~40,000 | 0% *(rare)* |

> The 50% savings figure is the **conservative** estimate. Most sessions save 90–98%.

---

## Installation

### 1. Copy the skill file

```bash
mkdir -p ~/.claude/skills
cp breadcrumb.md ~/.claude/skills/breadcrumb.md
```

### 2. Initialize for your project

Navigate to your project root and run:

```
/breadcrumb init
```

Claude will explore your codebase and generate:
```
your-project/
└── .claude/
    └── context/
        ├── system-map.md   ← architecture snapshot
        └── session-log.md  ← session history
```

### 3. Use it

```
/breadcrumb          → load context (run at start of every session)
/breadcrumb update   → save progress (run at end of session)
```

---

## The `system-map.md` Format

```markdown
# System Map — MyApp
> Last updated: 2026-03-27 | git: a3f8c12

## STACK
Next.js 14 App Router | Supabase (PostgreSQL + Auth + Realtime) | Vercel | Tailwind + shadcn

## ARCHITECTURE
PUBLIC (SSR/indexed): /, /about, /pricing, /blog/*
AUTH: /login, /register, /forgot-password
PRIVATE (CSR/auth-gated): /dashboard/*, /settings
MIDDLEWARE: middleware.ts → redirects unauthenticated /dashboard/* to /login

## FILE INDEX
app/dashboard/layout.tsx → root layout for all dashboard pages; loads profile + period context
lib/supabase/client.ts   → browser Supabase client; singleton via useRef to avoid re-init
hooks/useProfile.ts      → fetches user profile + currency pref; used on every dashboard page
lib/utils/amortization.ts → French amortization calc; estimatePaidInstallments() estimates paid count from balance

## ACTIVE FEATURES
Debts module: ✅ done | insurance_rate_mv column added (migration 074)
Credit card installments: 🚧 in progress | 8-phase feature, phases 1–2 complete
AI Insights: ✅ done | per-module, lazy-loaded, Supabase edge function

## KNOWN BUGS / DECISIONS
- debt.total_paid is unreliable: use estimatePaidInstallments() × base_payment instead (can drift if payments registered without reducing balance)
- Credit simulator restricted to super_admin: still in testing, not ready for general users
- Supabase CLI not configured: migrations applied manually in dashboard

## PENDING
- Apply migration 075 (credit card installments schema)
- Update insurance rate on Dairon's credit to 0.12417
```

**Rules for system-map:**
- **Max 800 tokens** — compress without mercy
- **One line per file** in FILE INDEX
- **No prose** — only facts
- **The KNOWN BUGS / DECISIONS section** is the most valuable: captures WHY decisions were made, which is invisible in the code

---

## The `session-log.md` Format

```markdown
# Session Log

## 2026-03-27 — debt insurance feature
Done: added insurance_rate_mv column, checkbox in DebtDialog, derived total_paid from amortization table
Files: types/database.ts, components/modules/debts/DebtDialog.tsx, app/dashboard/debts/page.tsx
Decisions: derive total_paid from estimatePaidInstallments() because debt.total_paid drifts when payments registered manually without updating balance
Next: update Dairon credit insurance rate to 0.12417, apply migration manually in Supabase
---

## 2026-03-25 — SEO audit
Done: full 7-agent parallel audit, report saved to docs/seo-audit-2026-03-25.md
Files: docs/seo-audit-2026-03-25.md
Decisions: robots.ts not blocking AI crawlers — blockage is at Cloudflare level, not in code. Fix is CF dashboard, not repo.
Next: fix /contact (server component), create llms.txt, add userScalable=false to layout.tsx
---
```

---

## Why This Works Better Than Just Taking Notes

| Property | breadcrumb | Plain notes |
|----------|-----------|-------------|
| Structured and parseable | ✅ Claude knows exactly where to look | ❌ Requires interpretation |
| Captures the WHY | ✅ Explicit Decisions section | ❌ Usually just describes what |
| Incremental updates | ✅ Edit only changed lines | ❌ Usually rewritten from scratch |
| Uses git as truth | ✅ Diff detects exactly what changed | ❌ Manual tracking |
| Lives with the code | ✅ `.claude/context/` is in the repo | ❌ External or separate |
| Survives team handoffs | ✅ Commit `.claude/context/` to git | ❌ Personal and ephemeral |

---

## Complementary to Claude's Memory System

breadcrumb and Claude's built-in memory system serve different purposes:

| System | Stores | Loaded |
|--------|--------|--------|
| `~/.claude/projects/.../memory/` | User preferences, feedback, collaboration style | Automatically, every session |
| `.claude/context/` (breadcrumb) | Codebase state, file index, technical decisions | On demand via `/breadcrumb` |

They are **complementary, not competing**. Memory knows *how to work with you*. breadcrumb knows *what the code is*.

---

## `.claude/context/` — Version Control Decision

| Option | When to use |
|--------|-------------|
| **Commit to git** | Team projects — breadcrumb becomes shared knowledge |
| **Add to `.gitignore`** | Personal/sensitive projects — keeps it local |

If you version it, every team member gets project context for free. New hires can run `/breadcrumb load` on day one and understand the architecture immediately.

---

## License

MIT — use freely in personal or commercial projects.

---

*Built for [Claude Code](https://claude.ai/code). Designed to make every session feel like you never left.*
