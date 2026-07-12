# Notes workflow — prompts for Claude Desktop

Reusable prompts for turning lecture screenshots into CKAD notes. Paste the
relevant one into Claude Desktop. Keep one chat per section; start a fresh chat
with the continuation prompt when you hit the 100-image limit.

## The one rule that makes images actually show up

Listing a diagram in YAML frontmatter (`companion_diagrams:`) is **metadata
only — it never renders**. An image appears in the notes only if it is embedded
in the body with a relative path:

```markdown
![Short caption](./diagrams/14-something.png)
```

Always `./diagrams/...` — never `../diagrams/...`, never repo-rooted paths like `notes/...`.

## Numbering rules

- Chapter files: `<CC>-<chapter-slug>.md`, one topic per file. Chapter numbers
  reset to `01` (or `00` for intro/setup) at the start of each section folder.
- Diagrams: `<NN>-<diagram-slug>.png` in the section's own `diagrams/` subfolder.
  Diagram numbers run continuously **within a section** — they do NOT reset per
  chapter, only per section.
- To resume: look in the section folder for the highest existing `CC-` file and
  the highest `NN-` in `diagrams/`, then add 1 to each.

---

## Generic section-start prompt

> You are helping me build CKAD study notes from lecture screenshots I upload. Follow these rules exactly.
>
> **File layout**
> - Notes live in `notes/<SECTION-FOLDER>/`.
> - Each lecture = one markdown file, one topic per file, named `<CC>-<chapter-slug>.md`.
> - Diagrams go in that section's own `<SECTION-FOLDER>/diagrams/` subfolder, named `<NN>-<diagram-slug>.png`.
>
> **Numbering (critical)**
> - Section folder: `<SECTION-FOLDER>`
> - Start chapter numbering at: `<START-CHAPTER>` and increment by 1 per lecture.
> - Start diagram numbering at: `<START-DIAGRAM>` and increment by 1 for every diagram, continuously across all lectures in this section (diagram numbers do NOT reset per chapter).
> - Track the current chapter and diagram counters yourself and tell me the next available numbers whenever I ask.
>
> **Diagram referencing (must do this every time)**
> - Every diagram you reference MUST be embedded in the body of the markdown with a relative path: `![Caption](./diagrams/<NN>-<slug>.png)`. Place it in the section where it's relevant.
> - Frontmatter alone does NOT render images — never rely on a `companion_diagrams:` list to display anything.
> - For each diagram you embed, give me an "image save list" at the end of your reply: the exact filename (`<NN>-<slug>.png`) and a one-line description of what that image/screenshot should contain, so I save the right file to the right name.
>
> **Style (lean — no scaffolding)**
> - Concise, exam-focused. No emojis, no marketing fluff, no padding.
> - Keep all exam-relevant YAML and `kubectl` commands.
> - Do NOT add "Why this is in CKAD" preambles, "Key takeaways"/"TL;DR", "Open threads" checklists, instructor narration, or personal anecdotes.
> - For fields with no imperative generator, show how to recall them with `kubectl explain <path>` instead of telling me to memorize.
> - End each chapter with a `## References` section of 2-4 canonical, verified `kubernetes.io` links.
>
> Confirm the section folder and the starting chapter/diagram numbers back to me, then wait for my first lecture's screenshots.
>
> For this section: `<SECTION-FOLDER>` = `<fill in>`, `<START-CHAPTER>` = `<fill in>`, `<START-DIAGRAM>` = `<fill in>`. First lecture: `<topic>`.

---

## Mid-section continuation prompt (fresh chat after the 100-image limit)

> Continuing CKAD notes for section `<SECTION-FOLDER>`. Same rules as before: one topic per file, `<CC>-<slug>.md`; diagrams in `./diagrams/<NN>-<slug>.png` and ALWAYS embedded in the body with `![Caption](./diagrams/<NN>-<slug>.png)` (frontmatter does not render images); concise and exam-focused, no emojis. At the end give me the image save list (filename + description). Resume at chapter `<NEXT-CHAPTER>` and diagram `<NN>`. Here are the next screenshots.

---

## Diagram conventions

- **Format:** PNG, ~1600-1800px wide.
- **Tool:** matplotlib (preferred) or raw SVG → cairosvg.
- **Style palette:** dark background `#0d1b2a`, accent magenta `#ff2e93` for titles, text `#e6e6e6`, panels `#16263a` or `#3a3a3a`, terminals `#0a1420`. Code/yellow `#ffd166`, notes/cyan `#7fd1ff` or `#3ec6e0`, success/green `#3fbf5f`, warn/red `#ff4d4d`, purple accent `#7a5cff`.
- **Earn the diagram:** only generate one that adds something YAML or prose can't (hierarchies, before/after, flow over time, conceptual mappings). Tables of values → use markdown, not an image.
- **Verify visually:** after rendering, view the PNG and check for text overflow, alignment, and overlap; re-render if needed.