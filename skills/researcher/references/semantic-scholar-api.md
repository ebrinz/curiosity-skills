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
