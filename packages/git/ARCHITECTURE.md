# ARCHITECTURE.md — On-disk formats

Referenced from `PACKAGE.md §1, §4`. Every byte layout here must be
implemented exactly — this is what makes `mygit` interoperate with real
Git.

## 1. Object types

Four object types, each a byte blob prefixed with a header before hashing:

```
"<type> <byte-length-of-content-as-ascii>\0<content>"
```

SHA-1 of that full string (header + NUL + content) is the object id
(oid), rendered as 40 lowercase hex chars. The whole string (header +
content) is then zlib-deflated and written to
`.git/objects/<oid[0:2]>/<oid[2:40]>`.

### 1.1 blob
Raw file content, verbatim. No metadata (name, mode, permissions) —
those live in the tree that points to it. Identical file content anywhere
in history hashes to the same blob and is stored once.

### 1.2 tree
Sorted list of entries, each:

```
"<mode> <name>\0<20-byte-raw-oid>"
```

concatenated with no separators between entries. Notes:
- `mode` is ASCII octal, no leading zero stripped oddly:
  `100644` (regular file), `100755` (executable file),
  `120000` (symlink), `040000` (subdirectory → another tree object),
  `160000` (gitlink / submodule commit).
- Entries are sorted **byte-wise by name**, with directory names treated
  *as if* suffixed with `/` for sort purposes (this affects ordering
  relative to files with a common prefix — get this wrong and your tree
  oid won't match real git's for the same content).
- The 20-byte oid here is **raw binary**, not hex — unlike the object
  header. This trips up naive implementations.

### 1.3 commit
Plain text, strict field order:

```
tree <oid>
parent <oid>            (0 for root commit, 1 normally, 2+ for merges, one line each)
author <name> <email> <unix-timestamp> <tz-offset>
committer <name> <email> <unix-timestamp> <tz-offset>
[gpgsig <...>]           (optional, multi-line, skip for v1)

<commit message, blank line above>
```

- Timestamp format: `1699999999 -0400` (unix seconds, space, `+`/`-`HHMM).
- Message may be multi-line; a trailing newline is conventional.

### 1.4 tag (annotated tags only — lightweight tags are just refs, §3)

```
object <oid>
type <commit|tree|blob|tag>
tag <tagname>
tagger <name> <email> <timestamp> <tz>

<tag message>
```

## 2. Loose objects vs packfiles

- **Loose**: one zlib-deflated file per object, as above. Simple, used
  for new/recent objects and during active work.
- **Packed**: many objects compressed together in one `.pack` file, with
  a companion `.idx` file for O(log n) lookup by oid. Objects inside a
  pack may be stored as **deltas** against another object in the same
  pack (either by full oid reference — "ref delta" — or by relative
  byte offset — "offset delta", the default for git-native packs).
  Full binary layout (header, delta encoding opcodes, idx v2 format,
  fan-out table, checksum trailer) is specified in `PROTOCOL.md §3`,
  since packs are primarily produced/consumed during network transfer
  and `gc`.
- Lookup order when resolving an oid: loose objects first, then each
  pack's idx (packs are typically newer-to-older).

## 3. The index (`.git/index`) — binary staging area

This is the file `add`, `status`, `commit`, and `checkout` all read/write.
Layout (all multi-byte integers big-endian):

```
Header (12 bytes):
  4 bytes   signature "DIRC"
  4 bytes   version (2, 3, or 4 — implement version 2)
  4 bytes   number of entries (N)

N entries, each (fixed part 62 bytes + variable name + padding):
  4 bytes   ctime seconds
  4 bytes   ctime nanoseconds
  4 bytes   mtime seconds
  4 bytes   mtime nanoseconds
  4 bytes   dev
  4 bytes   ino
  4 bytes   mode            (same encoding as tree entries, but 32-bit)
  4 bytes   uid
  4 bytes   gid
  4 bytes   file size
  20 bytes  oid (raw binary, blob this entry currently points to)
  2 bytes   flags: 1 bit assume-valid, 1 bit extended,
            2 bits stage (0-3, used during unresolved merges),
            12 bits name length (or 0xFFF if name is longer, use name
            length directly then)
  N bytes   entry path (NUL-terminated)
  padding   NUL bytes so total entry length is a multiple of 8
            (version 2 rule)

Extensions (optional, TREE / REUC / etc.) — skip for v1, but the parser
must tolerate their presence (read the 4-byte signature + 4-byte size,
skip that many bytes) so mygit doesn't corrupt indexes written by real git.

Trailer (20 bytes):
  SHA-1 checksum of everything above.
```

**Stages 1/2/3** (in the flags field) are how the index represents an
unresolved merge conflict: stage 1 = common ancestor, stage 2 = "ours",
stage 3 = "theirs". A file with entries at stage 2/3 (no stage 0) is an
unmerged conflict; `status` must report it as such and `commit` must
refuse until it's resolved (stage 0 present again after user edits +
`add`).

## 4. Refs

- `.git/refs/heads/<branch>`: file containing 40-char oid + `\n`.
- `.git/refs/tags/<tag>`: same, for lightweight tags. Annotated tags
  point at a tag object, which itself points at a commit.
- `.git/refs/remotes/<remote>/<branch>`: tracking refs, updated by
  `fetch`, never by local commits.
- `.git/HEAD`: either `ref: refs/heads/<branch>\n` (attached) or a raw
  40-char oid (detached HEAD state).
- `.git/packed-refs`: flattens many loose refs into one file for
  efficiency (`<oid> <refname>` per line, `^<oid>` line following an
  annotated tag line = the oid the tag's tag-object points to, i.e. the
  peeled ref). Loose refs in `refs/` take precedence over entries here
  with the same name — always check loose first.
- Ref resolution must support: full ref name, shorthand (`main` →
  `refs/heads/main`), `HEAD`, `HEAD~N`, `HEAD^N`, `<oid-prefix>`
  (unambiguous abbreviation, minimum 4 chars), `@{upstream}`. This is
  what `rev-parse` implements; every other command should resolve
  ref-like arguments through the same routine.

## 5. Config (`.git/config`, `~/.gitconfig`, `/etc/gitconfig`)

INI-like, case-insensitive section/key, case-sensitive subsection:

```ini
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
[user]
    name = Jane Dev
    email = jane@example.com
[remote "origin"]
    url = https://example.com/repo.git
    fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
    remote = origin
    merge = refs/heads/main
```

Resolution precedence for a given key: local (`.git/config`) overrides
global (`~/.gitconfig`) overrides system (`/etc/gitconfig`). Implement
`mygit config [--global|--system] <key> [<value>]` reading/writing this
exact format — many other commands (`user.name`, `user.email`,
`remote.<name>.url`, `branch.<name>.merge`) depend on it.

## 6. `.gitignore` matching

- Read `.git/info/exclude`, then `.gitignore` in every directory from
  repo root down to the file's directory, then global
  `core.excludesfile` if configured — patterns closer to the file win on
  conflict, and a later pattern in the same file overrides an earlier one.
- Glob semantics: `*` doesn't cross `/`, `**` does, leading `/` anchors to
  that directory, trailing `/` matches directories only, leading `!`
  negates (re-includes) a previously excluded path.
- Already-tracked files are **not** re-excluded by a later `.gitignore`
  entry (`status`/`add` must check the index, not just the ignore rules).

## 7. Reflog (`.git/logs/HEAD`, `.git/logs/refs/heads/<branch>`)

One line per ref update, appended (never rewritten in place):

```
<old-oid> <new-oid> <name> <email> <timestamp> <tz>\t<action>: <detail>
```

Used by `mygit reflog` and `HEAD@{N}` / `<branch>@{N}` resolution — this
is the safety net that makes `reset --hard` and rebase recoverable, so it
must be written by every command that moves a ref (`commit`, `checkout`,
`merge`, `rebase`, `reset`, `cherry-pick`, `fetch` for remote-tracking
refs).

## 8. Three-tree diff semantics (what `status`/`diff`/`add`/`checkout` actually do)

| Command | Compares | Effect |
|---|---|---|
| `diff` (no args) | working dir ↔ index | shows unstaged changes |
| `diff --staged` | index ↔ HEAD | shows staged changes |
| `status` | both of the above | "changes staged" / "changes not staged" |
| `add <path>` | working dir → index | copies file content into a blob object, updates index entry |
| `checkout -- <path>` | index → working dir | overwrites working file with index's blob |
| `checkout <commit> -- <path>` | HEAD(commit) → index → working dir | resets both |
| `commit` | index → HEAD | writes tree from index, new commit object, moves branch ref |
| `reset --soft <commit>` | moves HEAD only | index & working dir untouched |
| `reset --mixed <commit>` (default) | moves HEAD, resets index to match | working dir untouched |
| `reset --hard <commit>` | moves HEAD, resets index **and** working dir | destructive |

Internalizing this table resolves most ambiguity about "what should this
flag actually do" while implementing `COMMANDS.md`.
