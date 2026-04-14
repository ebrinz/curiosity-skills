# Researcher Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code research skill with three depth levels, adaptive tool selection, parallel subagent dispatch, and modular academic reference files.

**Architecture:** One core SKILL.md (~300 words) handles the research process, depth levels, topic classification, and synthesis. Two reference files (arxiv-api.md, semantic-scholar-api.md) are loaded on-demand when academic research is detected. Developed in project directory, symlinked to ~/.claude/skills/ for global availability.

**Tech Stack:** Claude Code skill (Markdown), arXiv REST API, Semantic Scholar API, curl

---

## File Structure

```
/Users/crashy/Development/skillz/researcher/
  SKILL.md                          # Core skill — process, depth, classification, synthesis
  references/
    arxiv-api.md                    # arXiv API reference — search, fetch, BibTeX
    semantic-scholar-api.md         # Semantic Scholar API — citations, impact, recommendations
```

After development, symlink:
```
~/.claude/skills/researcher/ -> /Users/crashy/Development/skillz/researcher/
```

---

### Task 1: Create SKILL.md — Core Research Skill

**Files:**
- Create: `/Users/crashy/Development/skillz/researcher/SKILL.md`

This is the main skill file. It must be self-contained for general web research and include pointers to reference files for academic research. Target ~300 words for token efficiency.

- [ ] **Step 1: Write SKILL.md with frontmatter**

```markdown
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
   - arXiv: `curl` to arXiv API (fallback: `WebSearch` with `site:arxiv.org`)
   - Citations: `curl` to Semantic Scholar API
   - Library docs: context7 MCP if available, else `WebFetch`
   - Local PDFs: `Read` tool
3. **Deduplicate** — track URLs/paper IDs seen
4. **Saturation check** — stop if >80% results already collected

Between rounds (deep/exhaustive): extract new terms, follow citation graphs, target gaps.

### Subagent Dispatch (deep/exhaustive)

Dispatch parallel subagents per search angle. Each returns:
- Source URL / paper ID
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
```

- [ ] **Step 2: Verify SKILL.md is under 500 words**

Run: `wc -w /Users/crashy/Development/skillz/researcher/SKILL.md`
Expected: under 500 words (target ~300, some overhead for tables/formatting is fine)

- [ ] **Step 3: Verify frontmatter is valid**

Check:
- `name` field uses only letters, numbers, hyphens
- `description` starts with "Use when..."
- `description` does NOT summarize the skill's process/workflow
- Total frontmatter under 1024 characters

---

### Task 2: Create references/arxiv-api.md

**Files:**
- Create: `/Users/crashy/Development/skillz/researcher/references/arxiv-api.md`

Adapted from the Hermes arxiv skill. Replace Hermes-specific tools (`web_extract`) with Claude Code equivalents (`WebFetch`, `Read`). Keep the arXiv API patterns, search syntax, BibTeX generation, and clean output parsing.

- [ ] **Step 1: Create references directory**

Run: `mkdir -p /Users/crashy/Development/skillz/researcher/references`

- [ ] **Step 2: Write arxiv-api.md**

```markdown
# arXiv API Reference

Search and retrieve academic papers via arXiv's free REST API. No API key needed.

## Quick Reference

| Action | Command |
|--------|---------|
| Search papers | `curl "https://export.arxiv.org/api/query?search_query=all:QUERY&max_results=5"` |
| Get by ID | `curl "https://export.arxiv.org/api/query?id_list=2402.03300"` |
| Read abstract | `WebFetch` on `https://arxiv.org/abs/2402.03300` |
| Read full paper | `WebFetch` on `https://arxiv.org/pdf/2402.03300` or `Read` for local PDFs |

## Search Syntax

| Prefix | Searches | Example |
|--------|----------|---------|
| `all:` | All fields | `all:transformer+attention` |
| `ti:` | Title | `ti:large+language+models` |
| `au:` | Author | `au:vaswani` |
| `abs:` | Abstract | `abs:reinforcement+learning` |
| `cat:` | Category | `cat:cs.AI` |
| `co:` | Comment | `co:accepted+NeurIPS` |

### Boolean Operators

```
all:transformer+attention              # AND (default with +)
all:GPT+OR+all:BERT                    # OR
all:language+model+ANDNOT+all:vision   # AND NOT
ti:"chain+of+thought"                  # Exact phrase
au:hinton+AND+cat:cs.LG               # Combined
```

## Sort & Pagination

| Parameter | Options |
|-----------|---------|
| `sortBy` | `relevance`, `lastUpdatedDate`, `submittedDate` |
| `sortOrder` | `ascending`, `descending` |
| `start` | Offset (0-based) |
| `max_results` | Count (default 10, max 30000) |

## Clean Output Parsing

```bash
curl -s "https://export.arxiv.org/api/query?search_query=all:QUERY&max_results=5&sortBy=submittedDate&sortOrder=descending" | python3 -c "
import sys, xml.etree.ElementTree as ET
ns = {'a': 'http://www.w3.org/2005/Atom'}
root = ET.parse(sys.stdin).getroot()
for i, entry in enumerate(root.findall('a:entry', ns)):
    title = entry.find('a:title', ns).text.strip().replace('\n', ' ')
    arxiv_id = entry.find('a:id', ns).text.strip().split('/abs/')[-1]
    published = entry.find('a:published', ns).text[:10]
    authors = ', '.join(a.find('a:name', ns).text for a in entry.findall('a:author', ns))
    summary = entry.find('a:summary', ns).text.strip()[:200]
    print(f'{i+1}. [{arxiv_id}] {title}')
    print(f'   Authors: {authors}')
    print(f'   Published: {published}')
    print(f'   Abstract: {summary}...')
    print(f'   PDF: https://arxiv.org/pdf/{arxiv_id}')
    print()
"
```

## BibTeX Generation

```bash
curl -s "https://export.arxiv.org/api/query?id_list=1706.03762" | python3 -c "
import sys, xml.etree.ElementTree as ET
ns = {'a': 'http://www.w3.org/2005/Atom', 'arxiv': 'http://arxiv.org/schemas/atom'}
root = ET.parse(sys.stdin).getroot()
entry = root.find('a:entry', ns)
if entry is None: sys.exit('Paper not found')
title = entry.find('a:title', ns).text.strip().replace('\n', ' ')
authors = ' and '.join(a.find('a:name', ns).text for a in entry.findall('a:author', ns))
year = entry.find('a:published', ns).text[:4]
raw_id = entry.find('a:id', ns).text.strip().split('/abs/')[-1]
cat = entry.find('arxiv:primary_category', ns)
primary = cat.get('term') if cat is not None else 'cs.LG'
last_name = entry.find('a:author', ns).find('a:name', ns).text.split()[-1]
print(f'@article{{{last_name}{year}_{raw_id.replace(\".\", \"\")},')
print(f'  title     = {{{title}}},')
print(f'  author    = {{{authors}}},')
print(f'  year      = {{{year}}},')
print(f'  eprint    = {{{raw_id}}},')
print(f'  archivePrefix = {{arXiv}},')
print(f'  primaryClass  = {{{primary}}},')
print(f'  url       = {{https://arxiv.org/abs/{raw_id}}}')
print('}')
"
```

## Common Categories

| Category | Field |
|----------|-------|
| `cs.AI` | Artificial Intelligence |
| `cs.CL` | Computation and Language (NLP) |
| `cs.CV` | Computer Vision |
| `cs.LG` | Machine Learning |
| `cs.CR` | Cryptography and Security |
| `stat.ML` | Machine Learning (Statistics) |

## Notes

- arXiv returns Atom XML — use parsing snippet above
- Rate limit: ~1 request per 3 seconds
- IDs: old format (`hep-th/0601001`) vs new (`2402.03300`)
- URLs: PDF `https://arxiv.org/pdf/{id}` — Abstract `https://arxiv.org/abs/{id}` — HTML `https://arxiv.org/html/{id}`
- Versioning: `1706.03762` = latest, `1706.03762v1` = specific version. Preserve version suffix in citations.
- Withdrawn papers: check `<summary>` for "withdrawn" or "retracted"
```

- [ ] **Step 3: Verify file was created**

Run: `ls -la /Users/crashy/Development/skillz/researcher/references/arxiv-api.md`
Expected: file exists

---

### Task 3: Create references/semantic-scholar-api.md

**Files:**
- Create: `/Users/crashy/Development/skillz/researcher/references/semantic-scholar-api.md`

Adapted from the Semantic Scholar section of the Hermes arxiv skill. Standalone reference for citation analysis, impact metrics, and paper recommendations.

- [ ] **Step 1: Write semantic-scholar-api.md**

```markdown
# Semantic Scholar API Reference

Citation data, impact metrics, paper recommendations, and author profiles. Free, no key needed (1 req/sec; 100/sec with API key).

## Quick Reference

| Action | Endpoint |
|--------|----------|
| Paper details | `GET /graph/v1/paper/{id}` |
| Citations of paper | `GET /graph/v1/paper/{id}/citations` |
| References from paper | `GET /graph/v1/paper/{id}/references` |
| Search papers | `GET /graph/v1/paper/search` |
| Recommendations | `POST /recommendations/v1/papers/` |
| Author search | `GET /graph/v1/author/search` |

Base URL: `https://api.semanticscholar.org`

## Paper IDs

Multiple formats accepted:
- arXiv: `arXiv:2402.03300`
- DOI: `DOI:10.1234/example`
- Semantic Scholar ID: `649def34f8be52c8b66281af98ae884c09aef38b`

## Get Paper Details

```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/arXiv:2402.03300?fields=title,authors,citationCount,referenceCount,influentialCitationCount,year,abstract" | python3 -m json.tool
```

## Citations OF a Paper (who cited it)

```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/arXiv:2402.03300/citations?fields=title,authors,year,citationCount&limit=10" | python3 -m json.tool
```

## References FROM a Paper (what it cites)

```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/arXiv:2402.03300/references?fields=title,authors,year,citationCount&limit=10" | python3 -m json.tool
```

## Search Papers

```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/search?query=GRPO+reinforcement+learning&limit=5&fields=title,authors,year,citationCount,externalIds" | python3 -m json.tool
```

## Paper Recommendations

```bash
curl -s -X POST "https://api.semanticscholar.org/recommendations/v1/papers/" \
  -H "Content-Type: application/json" \
  -d '{"positivePaperIds": ["arXiv:2402.03300"], "negativePaperIds": []}' | python3 -m json.tool
```

## Author Profile

```bash
curl -s "https://api.semanticscholar.org/graph/v1/author/search?query=Yann+LeCun&fields=name,hIndex,citationCount,paperCount" | python3 -m json.tool
```

## Useful Fields

`title`, `authors`, `year`, `abstract`, `citationCount`, `referenceCount`, `influentialCitationCount`, `isOpenAccess`, `openAccessPdf`, `fieldsOfStudy`, `publicationVenue`, `externalIds`

## Rate Limits

| Tier | Rate |
|------|------|
| No key | 1 req/sec |
| With API key | 100 req/sec |

## Notes

- Returns JSON — pipe through `python3 -m json.tool` for readability
- `externalIds` contains arXiv ID, DOI, etc. — useful for cross-referencing
- `influentialCitationCount` measures citations where this paper was important to the citing work (not just mentioned)
- For recommendations, `positivePaperIds` = papers like this, `negativePaperIds` = papers NOT like this
```

- [ ] **Step 2: Verify file was created**

Run: `ls -la /Users/crashy/Development/skillz/researcher/references/semantic-scholar-api.md`
Expected: file exists

---

### Task 4: Deploy — Symlink to ~/.claude/skills/

**Files:**
- Create symlink: `~/.claude/skills/researcher` -> `/Users/crashy/Development/skillz/researcher/`

- [ ] **Step 1: Create ~/.claude/skills/ directory if needed**

Run: `mkdir -p ~/.claude/skills`

- [ ] **Step 2: Create symlink**

Run: `ln -sf /Users/crashy/Development/skillz/researcher ~/.claude/skills/researcher`

- [ ] **Step 3: Verify symlink and skill discovery**

Run: `ls -la ~/.claude/skills/researcher/SKILL.md`
Expected: file is accessible through symlink

- [ ] **Step 4: Verify full structure**

Run: `find ~/.claude/skills/researcher -type f`
Expected:
```
/Users/crashy/.claude/skills/researcher/SKILL.md
/Users/crashy/.claude/skills/researcher/references/arxiv-api.md
/Users/crashy/.claude/skills/researcher/references/semantic-scholar-api.md
```

---

### Task 5: Smoke Test — Verify Skill Loads and Works

- [ ] **Step 1: Start a new Claude Code session and verify the skill appears**

Run: `claude --print-skills 2>/dev/null || echo "Check skill list manually in a new session"`

Verify `researcher` appears in the available skills list.

- [ ] **Step 2: Test quick depth — general web research**

In a new Claude Code session, ask:
> "Research what MCP (Model Context Protocol) is — quick"

Verify:
- Skill triggers
- Uses WebSearch (not academic APIs)
- Returns findings conversationally
- No subagents dispatched

- [ ] **Step 3: Test deep depth — academic research**

In a new Claude Code session, ask:
> "Research the current state of the art in GRPO reinforcement learning — deep"

Verify:
- Skill triggers
- Detects academic topic
- Reads arxiv-api.md and semantic-scholar-api.md references
- Dispatches parallel subagents
- Returns structured output with academic additions (paper count, key papers table)

- [ ] **Step 4: Test exhaustive depth**

In a new Claude Code session, ask:
> "Do an exhaustive survey of chain-of-thought prompting techniques"

Verify:
- Multiple search rounds
- Stops when saturation reached (>80% repeat results)
- Saves to file (should exceed 10 sources)
