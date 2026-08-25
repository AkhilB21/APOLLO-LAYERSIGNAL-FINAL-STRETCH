# Obsidian ↔ Claude Code ↔ LayerSignal — Implementation Roadmap

## Assumptions (stated explicitly — correct before building)

- "LS dashboard" = **LayerSignal**: your self-contained offline HTML dashboard, three-layer signal framework, v4.
- Reasoning layer = **Claude Code**, not Claude Desktop chat. Code gets raw filesystem access; chat doesn't, without extra plumbing.
- Claude Code requires an active Claude subscription (Pro / Max / Team / Enterprise) or Console API billing. Free tier excludes it.
- LayerSignal's exact data I/O (file input, localStorage, hardcoded array) isn't confirmed from memory alone. Phase 0 resolves this before any bridge code is written — it decides the entire Phase 3 design.
- The "instruction article" referenced in the transcript is **Andrej Karpathy's LLM-wiki pattern** (gist, April 2026): persistent markdown knowledge base maintained by an LLM, visualized in Obsidian, no vector DB. Reference: `gist.github.com/karpathy/442a6bf555914893e9891c11519de94f`. The CLAUDE.md below is written for your trading system specifically, not copied from a third-party template.

---

## Architecture

Three layers, one loop:

- **Knowledge** — Obsidian vault. Markdown, wikilinks, graph view. Source of truth for concepts.
- **Reasoning** — Claude Code. Reads the vault and LayerSignal, writes back to the vault.
- **Signal** — LayerSignal. Your existing offline HTML three-layer detector.

Loop: LayerSignal flags a stock → Claude Code checks the flag against vault concepts → writes a grounded note back to the vault → later, trade outcomes feed back in, closing the loop.

### Directory layout

Keep the vault and LayerSignal as siblings — don't nest one inside the other:

```
trading-brain/
├── vault/                          # Obsidian vault root — open this in Obsidian
│   ├── CLAUDE.md                   # instruction article — Phase 2
│   ├── raw/                        # drop sources here before ingest
│   ├── wiki/
│   │   ├── index.md                # master index, read first
│   │   ├── log.md                  # append-only ingest log
│   │   ├── concepts/               # vcp.md, rsi-divergence.md, layersignal-l1.md, ...
│   │   ├── entities/                # people, tools, tickers you track long-term
│   │   └── signals/                # one dated file per LayerSignal flag — Phase 3
│   ├── journal/
│   │   └── trades/                 # broker exports + edge-stat reports — Phase 3
│   └── .claude/
│       └── skills/
│           ├── ingest/
│           ├── analyze-signal/
│           └── trade-journal/
└── layersignal/                    # your existing dashboard project, unchanged
    ├── index.html
    └── data/                       # confirm actual path/format in Phase 0
```

Launch from the vault so `CLAUDE.md` auto-loads, and add LayerSignal as an extra directory:

```bash
cd trading-brain/vault
claude --add-dir ../layersignal
```

Skills in `vault/.claude/skills/` load automatically. CLAUDE.md files in `--add-dir` paths do **not** load by default — irrelevant here since all instructions live in `vault/CLAUDE.md`.

---

## Phase 0 — Prerequisites and audit

**Install:**
- Claude Code: `npm install -g @anthropic-ai/claude-code` (Node 18+ required), or a native installer — see `code.claude.com/docs/en/setup`.
- Obsidian: `obsidian.md`, free, offline-first.

**Confirm access:** Claude Code needs Pro, Max, Team, Enterprise, or API billing. Check before building anything else.

**Audit LayerSignal — do this with Claude Code itself, before writing any bridge skill:**

```bash
cd trading-brain/layersignal
claude
```

Ask it to answer three questions directly from the source:
1. How does the dashboard receive stock data — `<input type=file>`, a paste box, or a hardcoded array in the JS?
2. Does the v4 "historical signal/correction table" have an export function (JSON/CSV), or does it only render in-DOM?
3. Is watchlist/Holdings state persisted to `localStorage` only, or written to a file?

The answer to (2) and (3) decides Phase 3:
- **File or export exists** → Claude Code reads it directly. Simplest path, use this.
- **localStorage only, no export** → two options: (a) add a one-button "Export JSON" to LayerSignal — a few lines of JS, trivial; or (b) use Claude Code's Chrome integration to read the live DOM of an open LayerSignal tab and extract the current table on request, no code changes to LayerSignal at all.

Your existing **LayerSignal handoff brief** (referenced in your own project notes) is already a structured summary of the dashboard. Keep it — it's the first thing Phase 2 ingests.

**Data source check:** if you eventually want LayerSignal's *inputs* refreshed automatically, your existing `nse_scanner.py` and TradingView CSV pipeline (`fix_csv.py`) already own that path. No scraping needed — you already hold this data.

---

## Phase 1 — Scaffold the vault

```bash
mkdir -p trading-brain/vault/{raw,wiki/concepts,wiki/entities,wiki/signals,journal/trades,.claude/skills}
touch trading-brain/vault/wiki/index.md trading-brain/vault/wiki/log.md
```

In Obsidian: **File → Open folder as vault** → select `trading-brain/vault`.

Optional: install the **Obsidian Web Clipper** browser extension, point its default save location at `raw/`. One-click capture of articles/videos straight into the ingest queue — this is what let Aniket skip manual note-taking in the transcript.

---

## Phase 2 — CLAUDE.md (the instruction article)

This is the single highest-leverage file in the system. Keep it under ~150 lines — procedure belongs in skills, not here.

```markdown
# Trading Brain — Vault Instructions

## Role
Maintain this vault as a compounding knowledge base (Karpathy LLM-wiki
pattern). Every ingest adds to `wiki/`. Never replace it wholesale.

## Schema
- `wiki/index.md` — master index. Read first. Update on every ingest.
- `wiki/log.md` — append-only. One line per ingest: date, source, what changed.
- `wiki/concepts/` — one file per idea (`vcp.md`, `rsi-divergence.md`,
  `layersignal-l1-accumulation.md`). Atomic. Cross-link with [[wikilinks]].
- `wiki/entities/` — people, tools, tickers tracked long-term.
- `wiki/signals/` — one dated file per LayerSignal flag. Written only by
  `/analyze-signal`. Don't hand-edit.
- `journal/trades/` — broker exports and edge-stat reports, written only
  by `/trade-journal`.

## Ingestion rules
- Before creating a concept file, search `wiki/concepts/` for an existing
  match. Extend it — don't duplicate.
- Every concept file cites its source (filename or URL) and ingest date.
- If new material contradicts an existing concept file, keep both claims
  and flag the contradiction inline. Don't silently overwrite.
- Update `wiki/index.md` and append one line to `wiki/log.md` after every
  ingest, with no exceptions.

## LayerSignal ground truth
LayerSignal is this user's own three-layer signal detector, opened as an
additional directory alongside this vault. Use these definitions verbatim
when reasoning about its output — don't infer alternate meanings from
general market knowledge:

- **Layer 1 — weekly accumulation**: RSI divergence + volume contraction,
  weekly timeframe.
- **Layer 2 — momentum confirmation**: 3-day RSI stack reordering + moving
  average slope turning up.
- **Layer 3 — daily entry trigger**: RSI(21), RSI(36), RSI(56) all cross
  above 50 on the same day.

A stock is "signal-complete" only when L1 → L2 → L3 fired in that order.
A lone L3 with no prior L1/L2 is a weaker, unconfirmed signal — say so
explicitly whenever it applies. Never round this up to "signal-complete."

## Chart line convention
On this user's TradingView charts: red = 200 SMA, black = 150 SMA,
green = 50 SMA. Never confuse the red 200-SMA with a Supertrend line.

## Skills
- `/ingest [path]` — Karpathy-pattern ingest of anything new in `raw/`.
- `/analyze-signal [ticker]` — cross-reference a LayerSignal flag against
  `wiki/concepts/`, write to `wiki/signals/`.
- `/trade-journal [broker export path]` — compute edge stats, cross-check
  against `wiki/signals/` for which layer-combinations actually paid off.
```

**First three ingests**, in order: the LayerSignal handoff brief, the position-sizer bug-fix note, and the CARTRADE/HDFCSML250 chart observations that originally shaped the framework. These are hard-won and currently live only in your own memory/notes — exactly what this system exists to stop you from re-deriving.

---

## Phase 3 — Bridge skills

### 3a. `/ingest` — generic Karpathy-pattern ingest

`vault/.claude/skills/ingest/SKILL.md`:

```yaml
---
name: ingest
description: Ingest everything in raw/ into the wiki, Karpathy LLM-wiki pattern. Use when new sources have been dropped into raw/.
argument-hint: [optional filename]
---

## Files waiting in raw/
!`ls -la ${CLAUDE_PROJECT_DIR}/raw/ 2>/dev/null || echo "raw/ empty or not found"`

## Task
Ingest $ARGUMENTS, or every new file in raw/ if no argument given:

1. Read the source. For screenshots/charts, describe the pattern before filing.
2. Check wiki/concepts/ for an existing match — extend, don't duplicate.
3. Write or update the relevant concept/entity file(s). Cite source + date.
4. Cross-link new/updated files with [[wikilinks]].
5. Update wiki/index.md.
6. Append one line to wiki/log.md: `YYYY-MM-DD — ingested <source> — <what changed>`.
7. Leave the raw file in place. Don't delete it.
```

### 3b. `/analyze-signal` — the actual bridge

`vault/.claude/skills/analyze-signal/SKILL.md`. The `!` line is the bridge itself — it pulls LayerSignal's live output straight into the prompt before Claude sees anything. Point it at whatever path Phase 0 confirmed:

```yaml
---
name: analyze-signal
description: Cross-reference a LayerSignal flag against the vault's trading concepts and write a dated analysis note. Use when LayerSignal flags a stock, or when checking a ticker against the layer framework.
argument-hint: [ticker]
---

## Today's LayerSignal output
!`cat ${CLAUDE_PROJECT_DIR}/../layersignal/data/latest-signals.json 2>/dev/null || echo "no export found at that path — confirm the path from Phase 0, or paste the signal manually"`

## Task
For $ARGUMENTS:

1. Pull the ticker's current L1/L2/L3 status from the output above.
2. Load wiki/concepts/layersignal-l1-accumulation.md, layersignal-l2-momentum.md,
   layersignal-l3-trigger.md, and any other relevant concept file (vcp.md,
   trend-template.md, if ingested).
3. State plainly whether the signal is complete (L1→L2→L3) or partial.
4. Check wiki/signals/ and journal/trades/ for precedent — note anything
   that historically weakened this exact combination.
5. Write wiki/signals/YYYY-MM-DD-<TICKER>.md: signal state, supporting
   evidence, contradicting evidence, plain-language verdict. Analysis
   only — no position sizing, no buy/sell instruction.
6. Append one line to wiki/log.md.
```

If Phase 0 found no export and no live-DOM route, replace the `!` line with a plain instruction to accept a pasted description instead — the rest of the skill is unchanged.

### 3c. `/trade-journal` — the feedback loop

This is the highest-value piece: it closes the loop by checking which layer-combinations actually made money, not just which ones fired.

`vault/.claude/skills/trade-journal/SKILL.md`:

```yaml
---
name: trade-journal
description: Ingest a broker trade export, compute edge stats, and cross-check against wiki/signals/ to find which LayerSignal layer-combinations actually paid off. Use after exporting trades from your broker console.
argument-hint: [path to trade export CSV]
---

## Task
For the trade export at $ARGUMENTS:

1. Parse closed and open positions separately.
2. Compute: win rate, average win, average loss, expectancy per trade,
   and trades needed to reach a stated capital target at current
   expectancy.
3. For each closed trade, match ticker + entry date against
   wiki/signals/*.md from the same window.
4. Group results by signal state at entry — L1→L2→L3 complete, partial,
   or no LayerSignal note at all. Report expectancy per group.
5. Write journal/trades/YYYY-MM-<summary>.md with the numbers and the
   layer-combination breakdown.
6. Flag, don't fix: if a group has fewer than ~15 trades, say the result
   isn't statistically reliable yet rather than drawing a conclusion.
```

### Optional — MCP bridge

Skip this if Claude Code is your only surface — its native Read/Write/Edit tools already cover the vault once `--add-dir` points at it. Add an MCP server only if you also want **Claude Desktop chat** (not Code) to query the vault:

- Native route: install the `obsidian-local-rest-api` community plugin (ships its own MCP endpoint), then `claude mcp add --transport http obsidian https://127.0.0.1:27124/mcp/ --header "Authorization: Bearer <api-key>"`.
- Alternative: `mcp-obsidian` (MarkusPfundstein, `uvx mcp-obsidian`), same underlying REST API plugin.

---

## Phase 4 — Automation

Claude Code has three scheduling mechanisms. Pick by where your files live:

**Cloud routines** (`claude.ai/code/routines`, or `/schedule` in the CLI)
- Runs even with your machine off.
- No local file access — clones a fresh GitHub repo per run.
- Wrong fit here: both the vault and LayerSignal are local-only. Skip this.

**Desktop scheduled tasks** — recommended default
- Runs on your machine, needs it awake (or "Keep computer awake" enabled).
- Full local file access — reads the vault and LayerSignal directly, same as an interactive session.
- Minimum interval: 1 minute. Persists across restarts.

**`/loop` (CLI, session-scoped)**
- Same local access as Desktop tasks.
- Requires the terminal session to stay open the whole time.
- Fallback if you're CLI-only with no Desktop app.

**Setup (Desktop):** Sidebar → Routines → New routine → Local. Instructions field:

```
Run /analyze-signal for every ticker LayerSignal currently flags at L3.
Append results to today's wiki/log.md entry. If any ticker completed
L1→L2→L3 for the first time today, note it as high-priority at the top
of the log.
```

Schedule: Daily, set to your post-market-close review time. `Run now` once after creating it, approve the permission prompts, and future runs auto-approve the same tools.

**Push notifications:** out of scope for this roadmap, but the mechanism is either a `channels` webhook (`/docs/en/channels`) or a direct Telegram Bot API call added as a step in the scheduled task's own instructions — a Phase 4 extension once the core loop is stable, not a prerequisite for it.

---

## Phase 5 — Guardrails

- **Version control the vault.** It's plain markdown — `git init` in `vault/`, commit after each ingest session. Trivial to diff, trivial to restore.
- **Keep CLAUDE.md under ~200 lines.** If it grows, move procedure into skills or `.claude/rules/` — CLAUDE.md is for facts and rules Claude needs every session, not step-by-step logic.
- **If you ever add live scraping behind a login** (mirroring the MarketSmith pattern from the transcript, for any paid vendor), check that vendor's terms of service for automated access first. Not applicable to LayerSignal itself — it's entirely your own local dashboard.
- **Back up the vault.** Zip and sync, or push the git repo somewhere private. It's just files.

---

## Phase 6 — Beyond LayerSignal (later, not now)

The same vault and CLAUDE.md can eventually ground the rest of your stack: LC-LS Engine/Apollo's scoring logic, `nse_scanner.py`'s backtests, and the Parquet behavior-tracking dataset — one knowledge layer, multiple engines reading from it. Out of scope here. Note for later: LC-LS Engine's own architectural review already flagged look-ahead bias and missing volume signals — resolve those inside that project before wiring it into this loop, so the vault isn't ingesting a scoring engine's unresolved bugs as if they were settled concepts.

---

## Quick-start command sequence

```bash
# Phase 0
npm install -g @anthropic-ai/claude-code
cd trading-brain/layersignal && claude   # audit — answer the 3 questions above

# Phase 1
mkdir -p trading-brain/vault/{raw,wiki/concepts,wiki/entities,wiki/signals,journal/trades,.claude/skills}
touch trading-brain/vault/wiki/index.md trading-brain/vault/wiki/log.md
# Obsidian: File → Open folder as vault → trading-brain/vault

# Phase 2
# write vault/CLAUDE.md (content above)
# drop the LayerSignal handoff brief + position-sizer note into vault/raw/

# Phase 3
mkdir -p vault/.claude/skills/{ingest,analyze-signal,trade-journal}
# write each SKILL.md (content above)

cd trading-brain/vault
claude --add-dir ../layersignal
> /ingest
> /analyze-signal RELIANCE

# Phase 4
# Desktop app → Routines → New routine → Local → paste instructions → Daily → Run now
```
