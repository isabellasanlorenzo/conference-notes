# Conference Notes

A running log of conference talks, keynotes, and conversations — the ideas that stuck, the frameworks worth revisiting, and the questions still open.

This is a personal knowledge base, not a transcript archive. Notes are organized by event and filtered for signal: things that changed how I think, introduced a framework I hadn't encountered, or raised a question I'm still sitting with.

---

## What's in here

Each conference gets its own folder. Inside, each talk gets its own file. Notes follow a loose structure:

- **Speaker + talk title** — who said it and where
- **The core argument** — what the talk was actually about, in plain language
- **Key ideas** — frameworks, mental models, specific concepts worth keeping
- **Quotes worth keeping** — things said well enough to preserve
- **Open questions** — things the talk raised but didn't resolve
- **References** — tools, people, papers, or projects mentioned

Not every section will be filled for every talk. If a talk only had one idea worth keeping, that's what's there.

---

## Structure

```
conference-notes/
├── README.md
├── YYYY-conference-name/
│   ├── README.md              ← event overview, dates, themes
│   ├── talk-title-speaker.md
│   └── talk-title-speaker.md
└── YYYY-conference-name/
    └── ...
```

Example:

```
2025-rsaconference/
├── README.md
├── keynote-anna-foundational-security.md
└── panel-threat-intel-sharing.md
```

---

## Note format

Each talk file follows this template:

```markdown
# Talk title

**Speaker:** Name, Title, Organization  
**Event:** Conference Name, Year  
**Track / Session:** (if applicable)

---

## Core argument

One paragraph. What was the talk actually claiming?

---

## Key ideas

### Idea or framework name
Explanation. Why it matters. How it connects to other things.

### Another idea
...

---

## Quotes

> "Exact words if they were said well enough to preserve."

---

## Open questions

- Things the talk raised but didn't answer
- Tensions or disagreements I noticed
- What I'd want to dig into further

---

## References

- Person mentioned: [@handle](link) or brief description
- Tool / project: [Name](link)
- Paper or resource: [Title](link)
```

---

## Why this exists

Conference talks are perishable. You absorb something in the room, it feels obvious and urgent, and three weeks later you can't reconstruct why it mattered. This repo is an attempt to fix that — to force the kind of synthesis that makes information actually stick.

The notes here are mine. They're opinionated, incomplete, and filtered through my own context. If you've found this repo and something resonates (or you think I got something wrong), feel free to open an issue.

---

## Coverage

| Year | Conference | Talks |
|------|------------|-------|
| 2026 | BSides SF | Anna Westelius Keynote |

---

## A note on format

Notes are plain Markdown. No build step, no framework, no dependencies. The goal is longevity — files that will be readable in ten years without needing to remember what tool generated them.

---

## License

Notes and writing are my own. Speaker ideas and quotes are attributed to their sources. If you're a speaker and you'd like something removed or corrected, open an issue.
