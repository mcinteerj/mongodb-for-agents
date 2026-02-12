# How This Was Built

This entire repo was built in a single session by an AI agent and a human with opinions.

## The Cast

**Jake** — MongoDB Solutions Architect. The human. Provided domain expertise, made architectural decisions, caught inconsistencies, and kept the agents honest.

**Claude Opus 4** — The orchestrator. Running inside [Pi](https://github.com/mariozechner/pi-coding-agent), an open-source coding agent harness. Opus coordinated the whole build, delegating to specialist sub-agents and making editorial decisions along the way.

The sub-agents (also Claude, also running in Pi):

- **Rita** 🔍 — The researcher. Given a topic, she goes wide with web search, reads primary sources, and produces structured research documents. More on her below.
- **Peter** 📋 — The planner. Fed research outputs and constraints, he produced the phased execution plan with dependency graphs and parallelization strategy.
- **Bob** 🔨 — The builder. Took research outputs and built the actual markdown files. Wrote every reference doc, the SKILL.md, AGENTS.md, README, and MCP guide.
- **Ruth** 👀 — The reviewer. Read Bob's work alongside Rita's original research and flagged accuracy issues, missing content, and inconsistencies. Every domain reference went through her.

## How Rita Works

Rita ran **14 research tasks** across this build, producing **~120KB of structured research** that fed every reference file.

Her workflow for each task:
1. **Go wide** — multiple `brave-search` queries to find primary sources, blog posts, Stack Overflow threads, and GitHub repos. Averaged **~5 web searches per task**, sometimes up to 19 for broad topics.
2. **Go deep** — fetch and extract content from the most promising URLs using `trafilatura` and `firecrawl`. Averaged **~8 page fetches per task**, reading MongoDB official docs, driver docs, and community resources.
3. **Synthesize** — distill findings into a structured markdown document with sections, tables, code examples, and anti-patterns. Saved to `~/temp/agent/` for Bob to consume.
4. **Supplement with Perplexity** — for topics needing synthesis across many sources (agent ecosystem analysis, domain prioritization), she used `perplexity` for broader research sweeps.

Across all 14 tasks: **~100 web searches**, **~110 page fetches**, **~10 perplexity queries**, producing research docs ranging from 70 to 531 lines each.

Her output format was consistent: numbered sections, concrete code examples, comparison tables, anti-pattern callouts, and source attribution. This structure made it easy for Bob to build from and Ruth to verify against.

### What She Produced

| Research Task | Output Size | Key Sources |
|---|---|---|
| Agent structure patterns (MCP vs rules vs skills) | 180 lines | GitHub repos, agent docs, community posts |
| MongoDB domain prioritization | 315 lines | MongoDB docs, University, Stack Overflow |
| Cross-agent file format compatibility | 102 lines | Claude/Cursor/Codex/Gemini/Windsurf docs |
| MongoDB docs structure & navigation | 147 + 70 lines | mongodb.com/docs, llms.txt analysis |
| Schema design best practices | 531 lines | MongoDB manual, data modeling patterns |
| Indexing strategy | 411 lines | MongoDB manual, index reference |
| Aggregation pipelines | 440 lines | MongoDB manual, aggregation reference |
| Performance tuning | 352 lines | MongoDB manual, server status reference |
| Atlas Search | 471 lines | Atlas Search docs |
| Vector Search | 314 lines | Atlas Vector Search docs |
| Security & auth | 264 lines | MongoDB security docs |

## The Process

### Phase 0: Research & Scaffolding
Two parallel Rita instances kicked off:
1. **Agent structure patterns** — What do AI agents (Claude Code, Cursor, Codex, Gemini, Windsurf, Pi) actually benefit from? MCP servers, rules files, skills, CLIs?
2. **MongoDB domain mapping** — What are the key MongoDB domains where agents need help? What do they get wrong most often?

Key finding: agents don't lack *access* to MongoDB (the official MCP server handles that). They lack *judgment* — schema design decisions, index strategy, aggregation pipeline construction. The gap is an expertise layer, not a tooling layer.

A third Rita researched cross-agent file format compatibility — which filename conventions work across all major agents. Answer: `AGENTS.md` is the closest to universal.

Bob scaffolded the repo on GitHub while research ran in parallel.

### Phase 1: Skeleton & Shared Assets
Four parallel tasks:
- Bob wrote `AGENTS.md` (the rules file) with symlinks for Claude/Cursor/Windsurf
- Bob wrote `SKILL.md` — the routing file with mandatory trigger language and reference table
- Bob wrote the MCP setup guide
- Rita researched MongoDB docs structure (URL patterns, search strategies, whether `llms.txt` is useful — spoiler: it's 4.5MB of noise)

Jake made key decisions here:
- Single `AGENTS.md` at repo root, not a `rules/` directory
- Single skill with sub-references, not separate skills per domain
- Mandatory, verbose trigger language in SKILL.md to compel lazy agents to actually read it
- No custom CLIs — link to the official MCP server instead

### Phase 2: Domain References
Each domain went through a quality loop:

```
Rita (research) → Bob (build) → Ruth (review) → Bob (incorporate feedback)
```

**Tier 1** — all four ran in parallel:
- Schema Design
- Aggregation Pipelines  
- Indexing Strategy
- Performance Tuning

**Tier 2** — ran in parallel after T1 landed:
- Atlas Search
- Vector Search
- Security & Auth

**Docs Navigation** was built from Rita's MongoDB docs structure research.

Ruth caught real issues in every review:
- Double `/docs/` in URL patterns
- Missing `null` caveat for covered queries
- Incomplete optimizer rules in aggregation
- Missing WiredTiger eviction thresholds in performance
- Missing CSFLE automatic vs explicit modes in security

Bob incorporated all feedback. Then a final cross-repo Ruth review caught 5 more issues (inconsistent document size guidance across files, missing `totalKeysExamined` in AGENTS.md, etc.).

### Phase 3: Polish
Jake reviewed the output and caught:
- Inconsistent code block language tags (JSON blocks with JavaScript comments)
- Tool-specific references (brave-search, firecrawl) that wouldn't work for all agents
- Incorrect assumption that MongoDB docs were JS-rendered (they're actually SSR — curl works fine)
- Missing URL examples for mongosh, database tools, and Atlas CLI

Each fix was applied and pushed immediately.

## The Numbers

- **~25 sub-agent tasks** across Rita, Peter, Bob, and Ruth
- **8 domain reference files** + SKILL.md + AGENTS.md + README + MCP guide
- **~1,700 lines** of expert MongoDB guidance
- **1 session**, start to finish

## The Session Log

The full, unedited Pi session log is available at [`pi_session_log.html`](pi_session_log.html).

> GitHub doesn't render HTML files directly. To view: download the file and open it in your browser, or view it at [mcinteerj.github.io/mongodb-for-agents/how-this-was-built/pi_session_log.html](https://mcinteerj.github.io/mongodb-for-agents/how-this-was-built/pi_session_log.html) (if GitHub Pages is enabled).

## Tools Used

- **[Pi](https://github.com/mariozechner/pi-coding-agent)** — Open-source coding agent harness with sub-agent delegation
- **Claude Opus 4** (Anthropic) — The model behind all agents
- **GitHub CLI (`gh`)** — Repo creation and management
