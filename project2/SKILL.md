---
name: project2-skill
description: Build and test conventions for the sample content project.
---

# Project2 Skill

*(Replace with your own commands.)*

| Task | Command |
|---|---|
| Build the site | `npm run build` |
| Run rtest | `python rtest.py` |

## Standing assertions (owned by the Tester)

- Article numbers are contiguous - no gaps, no duplicates.
- The prev/next chain is unbroken end to end.
- Library, sitemap and search-index counts agree with the article count.
- Titles are within the house length limit.

These catch a renumbering mistake at build time, which is the failure mode that hurts most.

## Key locations

| Path | Purpose |
|---|---|
| `capsules/` | article content (bracket-PREFIX states) |
| `tickets/` | integration tickets (suffix states) |
| `src/` | site source - never hand-edit `dist/` |