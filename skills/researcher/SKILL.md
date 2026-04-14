---
name: researcher
description: Use when asked to research a topic, investigate a question, find information from multiple sources, or survey academic literature. Triggers on "research", "look into", "find out about", "what does the literature say", "survey", "investigate".
---

# Researcher

Structured research skill with three depth levels. Adapts tool selection to what's available in the environment.

## Depth Levels

User-specified takes priority. Default: `deep`. May suggest `quick` for single-fact lookups but state the downgrade.

| Level | Rounds | Subagents | Use for |
|-------|--------|-----------|---------|
| `quick` | 1 | No | Single facts, one-source lookups |
| `deep` | 2-3 | Yes, parallel | Multi-faceted topics, comparisons, technical decisions |
| `exhaustive` | 3+ until saturation | Yes, parallel | Literature surveys, comprehensive analysis |

## Process

1. **Determine depth** — from user request or default `deep`
2. **Classify topic** — general, academic, or both
3. **Execute search rounds** — per depth level
4. **Synthesize** — deduplicate, theme, flag contradictions
5. **Present or save** — conversational default; save to `research/YYYY-MM-DD-<topic>.md` if 10+ sources or user asks

### Topic Classification

- **Academic signals**: "papers", "literature", "state of the art", arXiv IDs, author names, conference names (NeurIPS, ICML, ACL)
- **General**: everything else
- **Both**: "what approaches exist for X", technical topics mixing blogs and papers

When academic detected → read `references/arxiv-api.md` and `references/semantic-scholar-api.md` for API patterns.

### Search Rounds

Each round:
1. **Generate queries** — 2-3 (quick), 4-6 (deep/exhaustive). Vary angles: terminology, synonyms, related concepts
2. **Execute** — adaptive tool selection:
   - Web search: `WebSearch`
   - Page content: `WebFetch` (fallback: Playwright MCP if available)
   - arXiv: `curl` to arXiv API — **note: arXiv covers physics/CS/math/stats only**. For non-STEM topics, skip arXiv and use Semantic Scholar or WebSearch instead.
   - Citations: `curl` to Semantic Scholar API — **free tier is 1 req/sec**. Space requests with ~1.5s delay. For burst searches, use WebSearch as fallback.
   - Library docs: context7 MCP if available, else `WebFetch`
   - Local PDFs: `Read` tool
3. **Deduplicate** — track URLs/paper IDs seen
4. **Saturation check** — stop if >80% results already collected

Between rounds (deep/exhaustive): extract new terms, follow citation graphs, target gaps.

### Subagent Dispatch (deep/exhaustive)

**Important: subagents cannot use WebSearch, WebFetch, or Bash** — those permissions don't propagate. Two approaches:

**Approach A (recommended):** Run all searches yourself in parallel (multiple WebSearch calls), then dispatch subagents to *analyze and synthesize* the collected results.

**Approach B:** Dispatch subagents for knowledge-based analysis of each search angle (they can reason from training data), then supplement with your own sourced searches to verify and add citations.

Each subagent returns:
- Key claims or facts
- Relevance to question
- Suggested follow-up angles

## Output Format

```markdown
## Research: [Topic]

### Summary
[2-3 sentences]

### Key Findings
- **[Theme]**: [finding with source]

### Contradictions / Open Questions
- [where sources disagree]

### Sources
- [numbered, with URLs/titles/authors]
```

**Academic additions:** paper count, key papers table (title, authors, year, citations), citation graph highlights.
