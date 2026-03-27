---
name: breadcrumb
description: >
  Context compression system for Claude Code. Loads full project understanding
  in ~300 tokens using a structured system-map + session-log instead of reading
  the raw codebase. Supports three commands: load (start of session), update
  (end of session), and init (first-time setup).
user-invocable: true
args:
  - name: command
    description: "load (default), update, or init"
    required: false
---

You are executing the **breadcrumb** skill. Read the `command` argument (default: `load`) and follow the corresponding instructions below. Do NOT deviate from the prescribed steps.

---

## Command: `load` (default — run at session start)

**Goal:** Understand the full project context spending as few tokens as possible.

### Steps

1. **Check for context files**
   - Look for `.claude/context/system-map.md` in the current working directory.
   - If the file does not exist, tell the user:
     > "No breadcrumb context found. Run `/breadcrumb init` once to initialize the system."
     Then stop.

2. **Read context files** (in this order, no skipping)
   ```
   .claude/context/system-map.md    ← architecture, file index, known decisions
   .claude/context/session-log.md   ← recent sessions, pending tasks
   ```

3. **Detect changes since last session**
   Run: `git log --oneline -1` to get current HEAD hash.
   Compare with the `git:` hash stored in system-map.md header.

   - If hashes match → no changes, skip to step 4 directly.
   - If hashes differ → run:
     ```
     git diff <stored_hash> HEAD --name-only
     ```
     Then read ONLY the files listed in the diff output that are relevant to understanding the project (skip lockfiles, auto-generated files, assets).

4. **Report to user** with a compact summary:
   ```
   ✅ Breadcrumb loaded
   📁 Project: [name from system-map]
   🔧 Stack: [stack line from system-map]
   📋 Last session: [date + one-line summary from session-log]
   🔄 Changed files: [N files] — [list if > 0, or "none since last update"]
   📌 Pending: [pending tasks from system-map or session-log]
   ```

5. **You are now ready.** Do not read any more files unless the user's task requires it.

---

## Command: `update` (run at end of session or after major changes)

**Goal:** Keep the system-map and session-log accurate and up to date.

### Steps

1. **Identify what changed this session**
   - Which files were read, edited, or created?
   - What decisions were made and why?
   - What tasks were completed?
   - What is left pending?

2. **Update `system-map.md`**
   - Edit ONLY the lines in FILE INDEX that correspond to files you touched.
   - Update ACTIVE FEATURES status if any feature changed state.
   - Add to KNOWN BUGS / DECISIONS if a non-obvious decision was made.
   - Update PENDING with next concrete steps.
   - Update the header: `> Last updated: [today's date] | git: [current HEAD hash]`
   - **Hard rule:** Keep the total file under 800 tokens. Remove or compress stale entries.

3. **Prepend a new entry to `session-log.md`** using this exact format:
   ```
   ## [YYYY-MM-DD] — [one-line session title]
   Done: [bullet list of completed tasks]
   Files: [comma-separated list of files touched]
   Decisions: [key decisions and their WHY — this is the most valuable part]
   Next: [concrete next steps]
   ---
   ```
   Keep at most the last **5 entries** in the log. Remove older ones.

4. **Confirm to user:**
   ```
   ✅ Breadcrumb updated
   📝 system-map.md — [N lines changed]
   📓 session-log.md — new entry added
   ```

---

## Command: `init` (run once when setting up breadcrumb for a project)

**Goal:** Explore the codebase and generate the initial context files.

### Steps

1. **Acknowledge and set expectations**
   Tell the user:
   > "Initializing breadcrumb. I'll explore the codebase and generate `.claude/context/system-map.md` and `.claude/context/session-log.md`. This will take a moment and use more tokens than a normal session — but you only need to do this once."

2. **Explore the project structure**
   Run these commands to build a picture of the project:
   ```bash
   # Top-level structure
   ls -la

   # Source files (skip node_modules, .git, dist, .next, build)
   find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.py" -o -name "*.go" \) \
     -not -path "*/node_modules/*" -not -path "*/.git/*" -not -path "*/.next/*" \
     -not -path "*/dist/*" -not -path "*/build/*" | head -80

   # Package info
   cat package.json 2>/dev/null || cat pyproject.toml 2>/dev/null || cat go.mod 2>/dev/null

   # Git status
   git log --oneline -5
   git rev-parse --short HEAD
   ```

3. **Read key files selectively**
   Based on the file list, read the most architecturally important files:
   - Entry points (main layout, app router, main.ts, index)
   - Config files (next.config, tsconfig, env.example)
   - Auth middleware
   - Core type definitions
   - Database schema or ORM models
   - Main API routes

   Skip: tests, generated files, lockfiles, assets.

4. **Generate `.claude/context/system-map.md`**
   Use this exact structure (keep total under 800 tokens):
   ```markdown
   # System Map — [Project Name]
   > Last updated: [YYYY-MM-DD] | git: [short hash]

   ## STACK
   [framework] | [database] | [hosting] | [css] | [key libs]

   ## ARCHITECTURE
   PUBLIC (SSR): [routes]
   AUTH: [routes or description]
   PRIVATE (CSR): [routes]
   MIDDLEWARE: [what it does in one line]

   ## FILE INDEX
   [path/to/file.ts] → [what it does | key facts | bugs if any]
   [only non-obvious files — skip boilerplate and auto-generated]

   ## ACTIVE FEATURES
   [feature name]: [emoji] [description] | next: [step]
   ✅ done | 🚧 in progress | 🔒 restricted | ⚠️ bug | ❌ broken

   ## KNOWN BUGS / DECISIONS
   - [Decision or bug]: [WHY — the reasoning not visible in code]

   ## PENDING
   - [concrete next steps]
   ```

5. **Generate `.claude/context/session-log.md`**
   Create the initial entry:
   ```markdown
   # Session Log

   ## [YYYY-MM-DD] — breadcrumb init
   Done: initial system-map generated from codebase exploration
   Files: .claude/context/system-map.md, .claude/context/session-log.md
   Decisions: breadcrumb initialized — system-map reflects codebase state at this commit
   Next: [copy PENDING from system-map]
   ---
   ```

6. **Create the directory if needed** and write both files to `.claude/context/`.

7. **Confirm to user:**
   ```
   ✅ Breadcrumb initialized
   📁 .claude/context/system-map.md — [N lines]
   📓 .claude/context/session-log.md — initial entry

   Usage:
     /breadcrumb        → load context at session start (~300 tokens)
     /breadcrumb update → save progress at session end
   ```

---

## Rules that apply to all commands

- **Never read files that haven't changed** unless the user's task requires it.
- **Compression over completeness** in system-map: one line per file, zero prose.
- **The WHY is the most valuable part** of both system-map decisions and session-log entries. Always capture it.
- **system-map.md must never exceed 800 tokens.** Trim aggressively.
- **session-log.md keeps only the last 5 entries.** Delete older ones on update.
- If `.claude/context/` does not exist, create it before writing files.
