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

## Writing Voice

Match the voice of existing posts (see openwrt-ap-hostname-sniffer as reference):

### Punctuation and formatting
- Use `-` (single hyphen) for asides, never em-dashes or double hyphens
- Use contractions (it's, doesn't, you've, won't)

### Tone
- Matter-of-fact, not instructional or essay-like
- Make direct claims ("it's always empty", "there's no fix for this"), don't hedge ("more likely", "arguably", "it could be")
- Use "we" for shared reader/author actions, "you" for addressing the reader, "I"/"my" only for specific personal experiences

### Structure
- No transition sentences ("After implementing these changes...", "Now that we've covered...")
- No meta-commentary ("In this section", "Let's explore", "It's worth noting", "As mentioned earlier")
- No summary conclusions ("In conclusion", "To summarize", "Overall") - end with the last piece of content
- No formulaic openings ("In today's...", "As we navigate...", "Let's dive into...")
- Section headers should be task-oriented ("The Script", "Surviving Firmware Upgrades"), not conceptual ("Understanding X", "The Importance of Y")

### Word choice
- Never use AI-flagged vocabulary: "delve", "tapestry", "landscape" (metaphorical), "leverage", "utilize", "robust", "seamless", "innovative", "transformative", "pivotal", "testament", "realm", "garnered", "showcasing", "nuanced", "multifaceted", "intricate", "foster", "streamline", "paradigm"
- No "not just X, but Y" or "it's not X - it's Y" constructions
- No rhetorical tricolons ("what X, what Y, and what Z")
- Prefer plain words: "use" not "utilize", "start" not "commence", "field" not "landscape", "is" not "serves as"

### Sentence rhythm
- Vary sentence length. Mix short asides ("If it appears, you're good.") with longer technical explanations
