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

**1. Paste without the auto-indent staircase.** When you paste a multi-line block (e.g. copied from the Kubernetes docs) with autoindent on, each line indents further than the last and the block turns into a staircase. Turn it off first:

```vim
:set paste     " paste raw, no auto-indent
" ...paste your block...
:set nopaste   " turn it back on so o/O still auto-indent
```

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

## Recommended `~/.vimrc` for the exam

```vim
set expandtab tabstop=2 shiftwidth=2 number
```

- `expandtab` — Tab key inserts spaces (YAML rejects tabs).
- `tabstop=2 shiftwidth=2` — 2-space indents, the Kubernetes YAML convention.
- `number` — show line numbers (so error messages like "line 23" are useful).

See `setup.md` for the one-liner to write this file at the start of the exam.
