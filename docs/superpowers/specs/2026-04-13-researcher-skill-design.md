# Researcher Skill Design

## Overview

A Claude Code skill for structured research — general web research and academic paper investigation — with user-specified depth levels, adaptive tool selection, and parallel subagent dispatch for deep work.

**Inspiration:** Adapted from NousResearch/hermes-agent research skills (arxiv, research-paper-writing), re-targeted for Claude Code's tool ecosystem.

## Skill Identity

```yaml
name: researcher
description: Use when asked to research a topic, investigate a question, find information from multiple sources, or survey academic literature. Triggers on "research", "look into", "find out about", "what does the literature say", "survey", "investigate".
```

## Depth Levels

User-specified takes priority. If not specified, default to `deep`. Claude may suggest `quick` if the question is clearly a single-fact lookup ("what year was X published?") but should not downgrade without stating so.

| Level | Rounds | Subagents | When to use |
|-------|--------|-----------|-------------|
| `quick` | 1 round | No (sequential) | Simple factual questions, single-source lookups |
| `deep` | 2-3 rounds | Yes, parallel | Multi-faceted topics, comparing approaches, technical decisions |
| `exhaustive` | 3+ rounds, until saturation | Yes, parallel | Literature surveys, comprehensive competitive analysis, state-of-the-art reviews |

## Research Process Flow

1. **Determine depth level** — from user request or default to `deep`
2. **Classify topic** — general web, academic, or both
3. **Execute search rounds** — per depth level
4. **Synthesize findings** — deduplicate, theme, flag contradictions
5. **Present or save report** — conversational by default, file for substantial research

### Topic Classification Signals

- **Academic**: "papers", "literature", "state of the art", "citations", arXiv IDs, author names, conference names (NeurIPS, ICML, ACL, etc.)
- **General**: everything else — tools, products, comparisons, how-tos, current events
- **Both**: "what approaches exist for X", technical topics where blog posts and papers both matter

## Search Round Mechanics

Each round:

1. **Generate queries** — 2-3 for quick, 4-6 for deep/exhaustive. Cover different angles (terminology, synonyms, related concepts)
2. **Execute searches** — adaptive tool selection:

| Need | Preferred Tool | Fallback |
|------|---------------|----------|
| General web search | WebSearch | — |
| Fetch page content | WebFetch | Playwright (if MCP available) |
| arXiv papers | Bash (curl to arXiv API) | WebSearch with `site:arxiv.org` |
| Citation data | Bash (curl to Semantic Scholar API) | WebFetch on Google Scholar |
| Library docs | context7 (if MCP available) | WebFetch on official docs |
| Local PDFs | Read tool | — |

3. **Collect & deduplicate** — track URLs/paper IDs already seen
4. **Assess saturation** — if >80% of results already in collection, stop expanding

### Between Rounds (deep/exhaustive only)

- Extract new terminology, authors, concepts discovered
- Generate follow-up queries targeting gaps
- For academic: follow citation graphs (who cited X, what does X cite)

### Subagent Dispatch (deep/exhaustive)

Each parallel subagent gets one search angle with instructions to return structured findings:
- Source URL / paper ID
- Key claims or facts
- Relevance to the original question
- Suggested follow-up angles

## Synthesis & Output

### Synthesis Process

1. **Deduplicate and rank sources** — by relevance, recency, authority (citation count for papers, domain authority for web)
2. **Identify themes** — group findings into 3-5 key themes or categories
3. **Note contradictions** — flag where sources disagree, don't silently pick a side
4. **Assess confidence** — what's well-supported vs. what has thin evidence

### Output Format (conversational by default)

```markdown
## Research: [Topic]

### Summary
[2-3 sentence overview of what was found]

### Key Findings
- **[Theme 1]**: [finding with source attribution]
- **[Theme 2]**: [finding with source attribution]
- ...

### Contradictions / Open Questions
- [where sources disagree or gaps remain]

### Sources
- [numbered list with URLs, paper titles, authors where applicable]
```

### When to Save to File

If the research spans 10+ sources or the user asks, save to `research/YYYY-MM-DD-<topic>.md` in the working directory.

### Academic Research Additions

When academic sources are involved, additionally include:
- Paper count surveyed
- Key papers table (title, authors, year, citations, relevance)
- Citation graph highlights (seminal papers, recent trends)

## File Structure

```
researcher/
  SKILL.md                          # Core skill (~300 words)
                                    # - Depth levels, process flow,
                                    #   topic classification, synthesis format
                                    # - Pointers to references when academic
  references/
    arxiv-api.md                    # arXiv REST API patterns
                                    # - Search syntax, pagination, BibTeX generation
                                    # - Clean output parsing snippet
    semantic-scholar-api.md         # Citation graphs, impact metrics, recommendations
                                    # - Paper details, citations, references
                                    # - Author profiles, paper search
```

### What Goes Where

- **SKILL.md** — everything Claude needs for general web research + decision logic for when to load academic references. Self-contained for quick/general research.
- **references/arxiv-api.md** — loaded only when academic research is detected. Adapted from Hermes arxiv skill, stripped of Hermes-specific tooling.
- **references/semantic-scholar-api.md** — loaded when citation analysis, impact assessment, or paper recommendations are needed.

## Deployment

- Develop in `/Users/crashy/Development/skillz/researcher/`
- Symlink to `~/.claude/skills/researcher` for immediate availability across all projects

## Future: Autonomous Agent

The research methodology defined in this skill can later power an autonomous agent (Agent SDK) for fire-and-forget research tasks. The skill is the brain; the agent is a different body. Build the skill first, extract to agent later if needed.
