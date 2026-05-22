# Project context for Claude

## What this repo is

CKAD (Certified Kubernetes Application Developer) study notes and lab setup.

- Notes live in `src/main/resources/ckad-notes/`, grouped into **per-section folders** named `NN-<section-slug>/` (e.g. `01-core-concepts/`, `02-configuration/`). Each section folder contains its own numbered chapter notes (`00-...`, `01-...`).
- Each section folder has its own `diagrams/` subfolder alongside the notes. Diagrams live with the section that references them — **not** in a single global folder.
- `src/main/resources/ckad-notes/commands.md` is the **global single-file command reference** (Ctrl+F across everything).
- `src/main/resources/ckad-notes/commands/` holds **per-topic focused command files** for quick navigation while studying one resource at a time.

### Notes file layout (the schema)

```
src/main/resources/ckad-notes/
├── 01-core-concepts/
│   ├── 00-local-lab-setup.md
│   ├── 01-architecture.md
│   ├── ...
│   └── diagrams/
│       ├── 01-cluster-overview.png
│       └── ...
├── 02-configuration/
│   ├── 01-container-images-docker.md
│   ├── 02-commands-and-arguments.md
│   └── diagrams/
│       ├── 01-dockerfile-anatomy.png
│       └── ...
├── commands/
└── commands.md
```

**Rules for new notes (apply this when generating chapters):**
- A new chapter goes in its section folder (`NN-<section>/CC-<chapter-slug>.md`). **Chapter numbers reset at each section** — every section's notes start at `01-...` (or `00-...` for an intro/setup file). Do *not* continue the numbering from the previous section.
- Diagrams referenced by a chapter go in that same section's `diagrams/` subfolder. Diagram numbers also reset per section — `02-configuration/diagrams/` starts at `01-...` independently of `01-core-concepts/diagrams/`.
- Image references in markdown use **relative paths from the notes file**: `![Caption](./diagrams/NN-name.png)`. Never use `../diagrams/...` and never hard-code `src/main/resources/...` paths — those break when the file moves.
- One topic per file. Don't bundle multiple course chapters into one markdown file.

---

## Commit & PR workflow (required for every change)

Every change — even a small notes edit — goes through this flow. Don't commit directly to `main`.

1. **Open a GitHub issue** describing what's being added or changed.
   ```bash
   gh issue create --title "Add notes for <topic>" --body "Covers <what>. Includes diagrams <N>-<M>."
   ```
   Capture the issue number from the output.

2. **Create a feature branch** off `main`:
   ```bash
   git switch -c notes/<chapter-slug>      # for new chapter notes
   git switch -c docs/<short-slug>         # for commands or meta docs
   git switch -c fix/<short-slug>          # for corrections
   ```

3. **Commit** on the branch with a clear, conventional message:
   ```bash
   git add <files>
   git commit -m "Add <topic> notes (chapters NN-MM)"
   ```
   No emojis. No co-author trailer unless explicitly asked. One concise subject line; longer body only if the change needs explanation.

4. **Push the branch** to `origin`:
   ```bash
   git push -u origin HEAD
   ```

5. **Open a PR** with `gh pr create`. The PR body **must** include a GitHub auto-close keyword referencing the issue, so merging the PR closes the issue automatically:
   ```bash
   gh pr create --title "Add <topic> notes" --body "$(cat <<'EOF'
   ## Summary
   - <what changed>

   Closes #<issue-number>
   EOF
   )"
   ```
   Auto-close keywords (any work, case-insensitive): `Closes #N`, `Fixes #N`, `Resolves #N`. The reference must be in the PR description or in a commit message that lands on the default branch.

6. **Merge the PR** (`gh pr merge --squash --delete-branch` is the default). The linked issue closes automatically when the merge reaches `main`.

### Quick sanity check before merging
- Issue linked in PR body via `Closes #N`? ✓
- Branch is feature branch, not `main`? ✓
- Commit message is informative (no "wip", no "update")? ✓

---

## Commands documentation rules

When a change introduces or surfaces new `kubectl`/`kind`/shell commands:

- **Always update `commands.md`** (the global reference). New commands belong in the relevant numbered section.
- **Always update the per-topic file** in `commands/` for the resource(s) touched.
- The dedicated **`commands/imperative.md`** is the exam-time reference for imperative one-liners + `--dry-run=client -o yaml` YAML generation — keep its examples in sync when new resources are covered.

If a notes chapter introduces a new resource type that has no focused command file yet, create one in `commands/` and add it to `commands/README.md` index.

---

## Style

- No emojis in notes or commit messages unless the user explicitly asks for them.
- Concise, exam-focused phrasing. No marketing fluff, no padding.
- Diagrams referenced by their numbered filename prefix from the **section's own** `diagrams/` subfolder, via `./diagrams/<NN-name>.png`.
- Prefer editing existing files over creating new ones; keep new files focused (one topic per file).
- When updating notes, also update `commands.md` and the matching `commands/<topic>.md` in the same PR so the references never drift.