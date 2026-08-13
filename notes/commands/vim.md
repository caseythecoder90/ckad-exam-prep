# Vim quick reference

You'll be in vim a lot on the exam — to edit generated YAML. These are the keystrokes that matter.

| Action | Command |
|---|---|
| Open file | `vim file.yaml` |
| Enter Insert mode (before cursor) | `i` |
| Leave Insert mode | `Esc` |
| Save | `:w` |
| Save and quit | `:wq` |
| Quit without saving | `:q!` |
| Jump to line N | `:N` |
| Go to top / bottom | `gg` / `G` |
| Search forward | `/word` then Enter (`n` for next, `N` for previous) |
| Delete current line | `dd` |
| Copy (yank) current line | `yy` |
| Paste below / above cursor | `p` / `P` |
| Undo / Redo | `u` / `Ctrl+r` |
| Indent / outdent line | `>>` / `<<` |
| Visual select line | `V` then move cursor |
| Replace word under cursor | `cw` then type new word |

### Insert-mode entry points (where you land matters)

The whole point: enter Insert mode at the *right spot* instead of `i` then arrowing around.

| Action | Command |
|---|---|
| **Open blank line *below* cursor** + insert | `o` |
| **Open blank line *above* cursor** + insert | `O` |
| Append *after* cursor | `a` |
| Append at *end of line* (e.g. add a value) | `A` |
| Insert at *first non-blank* of line | `I` |

`o` / `A` are the two you'll reach for constantly: `o` to add the next list item (a new `env`, `volumeMounts`, or label line), `A` to tack a value onto the end of the line you're on.

### Moving and editing without leaving Normal mode

| Action | Command |
|---|---|
| Start / end of line | `0` / `$` |
| Next / previous word | `w` / `b` |
| Delete char under cursor | `x` |
| Delete word / to end of line | `dw` / `D` |
| Change to end of line | `C` |
| Replace single char (stay in Normal) | `r<char>` |
| Delete N lines | `N` + `dd` (e.g. `3dd`) |
| **Duplicate current line** | `yyp` |
| Repeat last change | `.` |
| Join line below onto current | `J` |

`yyp` (yank line, paste below) is the fastest way to add a near-identical line — another label under `labels:`, another `- name:`/`value:` pair under `env:`. Duplicate, then edit the copy.

## YAML power moves (the ones that save real time)

YAML lives and dies by indentation, so most exam-time pain is fixing indent or pasting cleanly. These four solve 90% of it.

**1. Pasting a multi-line block — the two failure modes.**

*Paste mode OFF* → the **staircase**: autoindent re-indents each pasted line on
top of the indentation it already carries, so the block marches right.

```vim
:set paste     " paste raw, no auto-indent
" ...paste your block...
:set nopaste   " turn it back on so o/O still auto-indent
```

*Paste mode ON* → the **left-shift**, which surprises people because paste mode
was supposed to be the fix. Symptom: the first pasted line lands perfectly, every
line after it sits too far left.

```yaml
    terminationMessagePolicy: File
    startupProbe:        # line 1 — landed at the cursor, looks right
  httpGet:               # lines 2+ — column 0 + their own indent = too far left
    path: /
    port: 80
```

The rule that explains it: **paste mode inserts your clipboard verbatim — line 1
starts at the cursor, every later line starts at column 0** plus whatever
indentation it carried. Vim adds nothing, so only line 1 gets your cursor's indent.

The reliable recipe — make line 1 start at column 0 too, then move the block as a
unit (relative indentation inside the block is always preserved):

```
" 1. cursor on the line ABOVE where the block goes
o          " opens a new line at column 0 (paste mode = no auto-indent)
           " 2. paste
Esc
" 3. select the pasted block and shift it into place
V4j        " line-visual, extend down 4 lines (or V} to the end of the block)
2>         " shift right 2 levels (with shiftwidth=2 that's 4 columns)
```

`>` shifts right, `<` shifts left, `.` repeats the last shift. Copying your source
block flush-left makes this fully predictable: always paste at column 0, then
shift right N times.

> For a short block you've memorized — a probe, a `securityContext`, an env var —
> **type it with paste mode off** instead. Autoindent keeps each new line at the
> previous line's indent, so it just flows. Reserve pasting for blocks big enough
> that retyping is genuinely slower.

**2. Indent / outdent a whole block.** Visual-select the lines, then shift. Repeatable with `.`:

```
V        " line-visual mode
j j j    " extend selection down (or use } to end of block)
>        " indent the whole selection one shiftwidth
.        " repeat to indent again
```

**3. Insert the same text at the start of many lines (block insert).** This is how you indent or prefix a column of lines at once — great for nudging an `env:` list:

```
Ctrl+v   " visual BLOCK mode
j j j    " select down the column
I        " insert at start of block
<type spaces>
Esc      " change applies to ALL selected lines
```

**4. Search and replace across the file.** Rename an image, fix a typo'd key everywhere:

```vim
:%s/old/new/g      " every occurrence in the file
:%s/old/new/gc     " same, but confirm each (c = confirm)
:noh               " clear the search highlight when done
```

## Running kubectl without leaving vim

The exam desktop has `tmux` pre-installed and you can open more than one terminal, but for "check a field, then type it into the manifest" a split is usually the slower option — you still have to read across windows and retype. These three keep you in one window, and two of them put the text directly where you want it.

**1. Suspend and come back (`Ctrl+z` / `fg`).** The lowest-effort option, and the one to default to. `Ctrl+z` drops vim to the background and returns you to the shell; run anything; `fg` puts you straight back with the cursor where you left it.

```
Ctrl+z                       " vim -> background
k explain pod.spec.containers.resources
fg                           " back into vim, cursor unmoved
```

**2. Read command output *into* the buffer (`:r !cmd`).** Inserts stdout below the cursor. This is the one that replaces a split terminal — pull real YAML in and delete what you don't need, instead of reading and retyping:

```vim
:r !kubectl get deploy web -o yaml        " pull a live manifest into the file
:r !kubectl explain pod.spec.containers.resources
```

Position the cursor first — output lands on the following line, already at column 0, so expect to re-indent a pulled block (`V` select, then `>`).

**3. Look without inserting (`:!cmd`).** Runs the command, shows output, waits for Enter, returns to the unchanged buffer:

```vim
:!kubectl get pv,pvc
```

Use `:!` to *check* something (did the PVC bind?) and `:r !` to *harvest* something (give me the fields).

> `:r !kubectl explain ...` pastes the prose descriptions too, so it is usually more deleting than typing. Prefer it for `-o yaml` output; prefer `Ctrl+z` for reading `explain`.

## Recommended `~/.vimrc` for the exam

```vim
set expandtab tabstop=2 shiftwidth=2 number
```

- `expandtab` — Tab key inserts spaces (YAML rejects tabs).
- `tabstop=2 shiftwidth=2` — 2-space indents, the Kubernetes YAML convention.
- `number` — show line numbers (so error messages like "line 23" are useful).

See `setup.md` for the one-liner to write this file at the start of the exam.
