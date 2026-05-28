# CKAD Notes Workflow

> Drop this file in the project root (e.g., `CKAD-PREP/CKAD-NOTES-WORKFLOW.md`)
> and attach it to new Claude chats — saves having to re-explain the task each
> time the context window fills up.

## What this project is

Casey's CKAD study repository. Notes from the Mumshad Mannambeth
KodeKloud/Udemy course, expanded with depth beyond the instructor's coverage.
Used to study, then back up with labs (KodeKloud, killer.sh) and the actual
exam.

## Repo layout

```
CKAD-PREP/
└── src/main/resources/
    └── ckad-notes/
        ├── 01-core-concepts/
        │   ├── 00-local-lab-setup.md
        │   ├── 01-...md
        │   └── diagrams/
        │       ├── 01-....png
        │       └── ...
        ├── 02-configuration/
        │   ├── 01-container-images-docker.md
        │   ├── 02-commands-and-arguments.md
        │   ├── 03-environment-variables.md
        │   ├── 04-configmaps.md
        │   ├── 05-secrets.md
        │   ├── 06-docker-security.md
        │   ├── 07-security-contexts.md
        │   ├── 08-resource-requirements.md
        │   └── diagrams/
        │       └── NN-topic.png
        ├── 03-... (future sections)
        ├── commands/
        └── commands.md
```

Each section has notes in a numbered sequence plus a `diagrams/` subfolder.
Diagrams are PNG, numbered globally within the section (continuing the
sequence — chapter 8 diagrams start at 13, 14, 15 etc. because earlier
chapters claimed 1-12).

## Task per lecture

For each lecture, Casey will:

1. Watch the lecture.
2. Upload screenshots (slides, terminal outputs, diagrams from the instructor).
3. Provide a written summary of what the instructor covered, in his own words,
   plus any questions, points of confusion, or topics he wants more depth on.

Claude then:

1. Produces ONE markdown notes file for the lecture.
2. Produces PNG diagram(s) saved into the section's `diagrams/` folder.
3. Stages everything under `/home/claude/deliver/` and ships via
   `present_files`.

## Notes file conventions

- **Header block** at the top with section, course chapter, why it's in CKAD
  (examinable vs. background), and any companion files.
- **Tone:** technical, no fluff, no AI-isms. Talks to Casey as a peer engineer.
- **Depth:** beyond the instructor when it serves understanding (real-world
  context from Casey's work, kernel/protocol/API mechanics, version history,
  CKS forward references where relevant) — but stay scoped to what helps pass
  CKAD. Mark CKS-only deep dives clearly.
- **Worked YAML manifests** wherever the instructor showed them, with
  comments. If the instructor showed two manifests, show two.
- **Section structure** typically:
  1. What this concept is / why it exists
  2. The mechanics
  3. Worked example(s)
  4. Imperative shortcuts (`kubectl create`, `--dry-run=client -o yaml`)
  5. Exam-pattern gotchas
  6. **TL;DR / takeaways** at the bottom
  7. **Resolved threads** + **Open threads** linking forward/back to other
     chapters
- **Real-world context** woven in where it earns its place (e.g., Casey's
  Visa node-failure incident in the ReplicaSet chapter). Don't force it.
- **Length:** detailed enough to study from cold, short enough to re-skim
  in 10 minutes. Roughly 300–800 lines depending on topic density. If a
  topic genuinely demands more (Secrets did), it gets more — but no padding.

## Diagram conventions

- **Format:** PNG, ~1600–1800px wide.
- **Tool:** matplotlib (preferred) or raw SVG → cairosvg, both work.
- **Style:** dark background `#0d1b2a`, accent magenta `#ff2e93` for titles,
  text `#e6e6e6`, panels `#16263a` or `#3a3a3a`, terminals `#0a1420`.
  Code/yellow `#ffd166`, notes/cyan `#7fd1ff` or `#3ec6e0`, success/green
  `#3fbf5f`, warn/red `#ff4d4d`, purple accent `#7a5cff`.
- **Naming:** `NN-topic-slug.png` where NN continues the section's running
  count. Casey occasionally drops a `bytes-and-cpu-units-reference.md` style
  reference card in the diagrams folder — fine.
- **Earn the diagram:** only generate diagrams that add something the YAML
  or prose can't. Visual hierarchies, before/after, flow over time,
  conceptual mappings — yes. Tables of values — no, use markdown.
- **Verify visually:** after rendering, view the PNG and check for text
  overflow, alignment issues, overlapping elements. Re-render if needed.

## Deliverable format

Always end the turn by calling `present_files` with the markdown file first
and the diagram(s) after. Tell Casey explicitly where each file goes in the
repo:

```
src/main/resources/ckad-notes/<section>/
├── <NN-topic>.md            ← new
└── diagrams/
    ├── <NN-diagram>.png     ← new
    └── ...
```

Brief written summary after `present_files` — what's in the chapter, what
weight specific diagrams got and why, any open threads carried forward.

## What NOT to do

- Don't produce content fluff just to make the notes longer.
- Don't repeat the instructor verbatim — Casey watched the lecture; the
  notes are for reinforcement and depth, not transcription.
- Don't generate decorative diagrams. Cut anything that's just text in a
  box.
- Don't break the file naming or folder layout — it's load-bearing.
- Don't switch to SVG-only output (Casey dropped SVGs after chapter 1).
- Don't add closing summaries about what Claude built. The files speak for
  themselves; one short orientation paragraph is enough.

## Future automation

Casey may connect the git repo to Claude to skip re-sharing the file
structure. Until then, he'll paste a `ls` of the section folder when needed.
