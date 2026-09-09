# COMMANDS.md — CLI surface

Referenced from `PACKAGE.md §5`. Format conventions assumed here are
defined in `ARCHITECTURE.md`. "Porcelain" = user-facing commands.
"Plumbing" = low-level building blocks porcelain is built on top of;
scripts rely on plumbing having stable, greppable output.

## Part A — Porcelain commands

### Setup

**`mygit init [<dir>]`**
Creates the `.git` directory layout from `PACKAGE.md §4` in `<dir>`
(default cwd). Sets `HEAD` to `ref: refs/heads/main\n`. Idempotent — safe
to re-run on an existing repo (does not destroy objects/refs).

**`mygit clone <url> [<dir>]`**
1. Determine transport from `<url>` scheme (`file://`, `https://`,
   `ssh://`, or a bare local path). See `PROTOCOL.md`.
2. `init` a new repo in `<dir>` (default: basename of url minus `.git`).
3. Negotiate and fetch all objects reachable from the remote's refs.
4. Write `refs/remotes/origin/*` for every remote branch, set
   `remote.origin.url`, `remote.origin.fetch`.
5. Create local `refs/heads/<default-branch>` pointing at the same oid as
   the remote's HEAD, set local `HEAD` to it.
6. Checkout that commit into the working directory (populate index +
   working files).

**`mygit config [--global] <key> [<value>]`**
Get (no value given) or set a config key as `section.subsection.key`.
Exit 1 if getting a key that doesn't exist.

### Basic snapshotting

**`mygit add <pathspec>...`**
For each matched file: hash-object it into the object db, update its
index entry (oid, mode, stat info). Supports `-A` (all tracked+untracked
respecting ignore), `-p`/`--patch` (interactive hunk staging — requires
the diff engine, implement after `diff` exists), `-u` (update only
already-tracked paths).

**`mygit status [--short|-s] [--porcelain]`**
Long form: human-readable sections "Changes to be committed",
"Changes not staged for commit", "Untracked files", each listing paths
with a state word (`modified:`, `new file:`, `deleted:`).
`--porcelain`: stable 2-char-code + path format for scripts, e.g.
`M  file.txt` (staged modify), ` M file.txt` (unstaged modify),
`??  file.txt` (untracked), `UU file.txt` (unmerged). Column 1 = index
state, column 2 = worktree state.

**`mygit diff [<commit>] [--staged] [-- <pathspec>]`**
Unified diff format (`--- a/path`, `+++ b/path`, `@@ -l,s +l,s @@`
hunks) using Myers diff over line-tokenized content. See semantics table
in `ARCHITECTURE.md §8`. `--stat` prints a summary (files changed,
insertions/deletions bar) instead of the patch.

**`mygit commit [-m <msg>] [--amend] [-a]`**
Refuses (exit 1) if index has unresolved conflict stages, or if index ==
HEAD's tree (nothing to commit) unless `--allow-empty`. Without `-m`,
opens `$EDITOR` on `.git/COMMIT_EDITMSG` (pre-populated with a `status`
style comment block); empty message after stripping comments aborts the
commit. `-a` stages tracked-file modifications before committing (does
not add new untracked files). `--amend` builds a new commit reusing the
current HEAD's parent, replacing HEAD's oid on the branch ref, and by
default reuses the previous message unless `-m`/edited.
Writes the tree from the current index (`write-tree` plumbing), creates
the commit object, moves the current branch ref, appends to reflog.

**`mygit rm <path>...`** / **`mygit mv <src> <dst>`**
`rm`: removes from working dir and index (use `--cached` to unstage-only,
keep working file). `mv`: renames in working dir, updates index entry
path — note git does not store renames explicitly; `diff`/`log --follow`
detect renames heuristically by content similarity (≥50% by default) at
diff time, not at commit time. Implement rename detection in the diff
engine, not as stored metadata.

### Branching & history navigation

**`mygit branch [<name>] [-d|-D <name>] [-m <old> <new>]`**
No args: list branches, `*` marks current. `<name>`: create branch ref
at current HEAD. `-d`: delete, refuses if not merged into HEAD unless
`-D`. `-m`: rename (moves the refs/heads file and updates
`branch.<name>.*` config).

**`mygit checkout <branch>`** / **`mygit switch <branch>`**
Updates HEAD to the target branch, replaces index + working dir with the
target's tree (refuses if doing so would overwrite uncommitted changes —
compare working dir against current index/HEAD first, exit 1 with a
clear conflict list if unsafe). `checkout <commit>` with no branch name
→ detached HEAD (HEAD becomes a raw oid). `checkout -b <new>` /
`switch -c <new>`: create-and-switch. `switch` is the newer, narrower
subset of `checkout` (branch switching only, no file-restore mode) —
implement `checkout -- <path>` (restore mode) as file-restore only, and
`checkout <ref>` as branch/detach only, matching real Git's UX split.

**`mygit log [--oneline] [--graph] [-n <N>] [<revrange>]`**
Walk commit parent pointers from the resolved starting ref (default
HEAD), most recent first. `--oneline`: `<short-oid> <first line>`.
`--graph`: ASCII `*`/`|`/`\`/`/` topology for merges. `<revrange>` syntax:
`A..B` (commits reachable from B, not from A), `A...B` (symmetric
difference). This requires implementing reachability (ancestor walk with
a visited-set) as a shared primitive — reuse it for `merge-base` too.

**`mygit show <ref>`**
Commit: print the commit header + the diff introduced (parent's tree vs
this tree). Blob: print raw content. Tag: print tag object then the
tagged object.

**`mygit blame <path>`**
For each line, walk back through commit history finding the last commit
that changed that line (standard line-tracking algorithm: start at HEAD,
diff each commit against its parent, attribute unchanged lines forward).
Print `<short-oid> (<author> <date> <line-no>) <content>` per line.

### Merging history

**`mygit merge <branch>`**
1. Find merge-base = lowest common ancestor of HEAD and `<branch>` (BFS/
   generation-number walk over parent graph).
2. If merge-base == `<branch>`'s tip: already up to date, no-op.
3. If merge-base == HEAD: **fast-forward** — just move the branch ref
   (and HEAD) to `<branch>`'s tip, update working dir/index to match, no
   new commit.
4. Otherwise: **three-way merge** — for every path, diff
   (base→ours) and (base→theirs); if only one side changed, take it; if
   both changed identically, take it; if both changed differently,
   **conflict**: write conflict markers into the working file:
   ```
   <<<<<<< HEAD
   ours content
   =======
   theirs content
   >>>>>>> branch-name
   ```
   and leave that path at index stages 1/2/3 (`ARCHITECTURE.md §3`).
   If no conflicts, auto-create a merge commit with two parents
   (HEAD, `<branch>` tip). If conflicts exist, stop and let the user
   resolve + `add` + `commit` (a plain `commit` with a pending merge
   detects `.git/MERGE_HEAD` and auto-fills two parents).

**`mygit rebase <upstream>`**
1. Compute the commit list unique to HEAD since merge-base(HEAD,
   upstream) (i.e. `merge-base..HEAD`), oldest first.
2. Move HEAD to `<upstream>`'s tip (detached).
3. Cherry-pick each commit from the list in order onto the new HEAD
   (reuse the cherry-pick plumbing below). On conflict: stop, leave
   `.git/rebase-merge/` state describing progress, let user resolve +
   `rebase --continue` (re-run cherry-pick's finalize step) or
   `--abort` (restore original HEAD from the saved oid, delete state
   dir).
4. When all commits are replayed, move the original branch ref to the
   new HEAD tip and reattach HEAD to that branch.
Implement as literally "rebase = repeated cherry-pick" — do not build a
separate merge algorithm for it.

**`mygit cherry-pick <commit>`**
Three-way merge of (`<commit>`'s parent tree → `<commit>`'s tree) applied
onto current HEAD tree, same conflict mechanics as `merge` step 4. On
success, creates a new commit with the original message but a new
parent (current HEAD) and new author-date-preserved/committer-date-now
metadata.

**`mygit revert <commit>`**
Same as cherry-pick but with the diff direction inverted (`<commit>`'s
tree → `<commit>`'s parent tree), producing a commit that undoes it.

**`mygit reset [--soft|--mixed|--hard] [<commit>]`**
See semantics table `ARCHITECTURE.md §8`. Default mode is `--mixed`.

**`mygit stash [push|pop|list|drop] [-m <msg>]`**
`push`: commit the current index+worktree diff (relative to HEAD) as two
special commit objects (one for index state, one for worktree-on-top-of-
index) referenced by `refs/stash` (a ref, with the previous stash as
that commit's parent — `refs/stash` is effectively a LIFO stack encoded
as a commit chain), then hard-reset to HEAD. `pop`: three-way merge the
top stash entry back into working dir + index, drop it from the stack on
success. `list`: walk the `refs/stash` parent chain.

**`mygit tag [<name>] [-a -m <msg>] [-d <name>]`**
No args: list tags. `<name>` alone: lightweight tag (plain ref).
`-a -m`: annotated tag (creates a tag object, `refs/tags/<name>` points
at it). `-d`: delete the ref.

### Remotes & networking (full protocol detail: `PROTOCOL.md`)

**`mygit remote add <name> <url>`** — writes `remote.<name>.url` +
default `fetch` refspec to config.
**`mygit fetch [<remote>]`** — negotiate + download new objects, update
`refs/remotes/<remote>/*`. Does not touch working dir or local branches.
**`mygit pull [<remote> [<branch>]]`** — `fetch` then `merge` (or
`rebase` if `pull.rebase` is set) the corresponding remote-tracking ref
into the current branch.
**`mygit push [<remote> [<branch>]]`** — negotiate + upload objects the
remote is missing, then request the remote update its ref. Reject
(non-fast-forward error) unless `--force`/`--force-with-lease` if the
remote ref isn't an ancestor of what's being pushed.

### Maintenance

**`mygit gc`** — pack all loose objects reachable from any ref into a
new packfile (`PROTOCOL.md §3`), delete now-redundant loose objects,
delete stash/reflog entries older than the configured expiry.
**`mygit fsck`** — walk every object, verify its hash matches its
content, verify every oid a tree/commit references actually exists;
report dangling (unreferenced) objects and any corruption.
**`mygit reflog [show]`** — print `.git/logs/HEAD` newest-first as
`<short-oid> HEAD@{N}: <action>: <detail>`.

## Part B — Plumbing commands

These have stable, minimal, script-friendly output and back every
porcelain command above.

| Command | Behavior |
|---|---|
| `hash-object [-w] <file>` | Compute the blob oid for a file; `-w` also writes it to the object db. Print the oid. |
| `cat-file (-t\|-s\|-p) <oid>` | `-t` prints type, `-s` prints size, `-p` pretty-prints content (deflate + parse). |
| `update-index --add <file>` | Insert/update one index entry directly. |
| `ls-files [--stage]` | List index entries; `--stage` includes mode/oid/stage-number. |
| `write-tree` | Serialize the current index into a tree object (recursively creating subtree objects for directories); print its oid. |
| `read-tree <oid>` | Populate the index from a tree object's contents. |
| `commit-tree <tree-oid> [-p <parent-oid>]... -m <msg>` | Create a commit object directly; print its oid. |
| `rev-parse <ref-expr>` | Resolve any ref expression (`ARCHITECTURE.md §4`) to a 40-char oid. |
| `symbolic-ref HEAD [<target>]` | Get/set what HEAD points to. |
| `update-ref <ref> <oid>` | Set a ref's oid directly, appending a reflog line. |
| `ls-tree <tree-oid>` | List a tree object's immediate entries (mode, type, oid, name). |
| `merge-base <a> <b>` | Print the oid of the lowest common ancestor. |
| `pack-objects` / `index-pack` / `unpack-objects` | Pack format read/write — see `PROTOCOL.md §3`. |

## Part C — Interaction conventions

- Read from a pager (`less`) for `log`/`diff`/`show` when stdout is a
  TTY and output exceeds one screen; never page when stdout is piped.
- Colorize add/remove lines in diffs and status categories when stdout
  is a TTY (`core.color=auto` semantics); strip color when not.
- Every destructive command that would discard uncommitted work
  (`checkout`, `reset --hard`, `clean`) must refuse by default and
  require an explicit force flag or a clean working tree — this is the
  single most load-bearing safety property of real Git's UX; do not
  relax it for convenience.
