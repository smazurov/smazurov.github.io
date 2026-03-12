# mazurov.dev

Hugo site using PaperMod theme. Deployed via GitHub Pages.

## Build

```bash
hugo --gc
```

## Content Guidelines

When writing new blog posts:

- Add `description` to front matter (concise, factual summary for schema and llms.txt)
- Optionally add `takeaways` list to front matter for key points rendered before content
- Include specific, verifiable statistics with sources when possible
- Quote authoritative sources directly where relevant
- Cite primary sources: RFCs, source code, vendor docs
- Lead first paragraph with the problem statement, not personal anecdote
- Include tested-on metadata (versions, hardware) for technical posts
- End posts with FAQ section when applicable
