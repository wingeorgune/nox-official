# PACKAGE.md — Building `mygit`: a reproduction of Git

## 0. Purpose

This is the entrypoint spec for an autonomous agent tasked with building a
functionally equivalent reimplementation of **Git**, the distributed version
control system originally written by Linus Torvalds (2005) and now
maintained by the Git community. The goal is not a toy — it is a
byte-compatible client: repositories produced by `mygit` must be readable by
real `git`, and vice versa.

Read this file first. It links to three companion specs which must also be
read in full before implementation begins:

- `ARCHITECTURE.md` — on-disk object model, index format, refs, config.
- `COMMANDS.md` — full CLI surface (porcelain + plumbing), flags, output.
- `PROTOCOL.md` — network transport (fetch/push), packfiles, negotiation.

## 1. Non-negotiable design constraints

These are what make output byte-compatible with real Git. Do not deviate:

1. **Content-addressable storage.** Every object is identified by the
   SHA-1 hash (hex, 40 chars) of `"<type> <size>\0<content>"`. (Modern Git
   also supports SHA-256 repos; implement SHA-1 first, it is the default
   and the one 99% of existing repos use.)
2. **Objects are zlib-deflated** (`zlib`/`miniz` compression, level default)
   when stored loose on disk.
3. **Three trees.** Every operation is explainable as movement of state
   between: the **working directory**, the **index** (staging area), and
   **HEAD** (last commit). Internalize this model before writing any code —
   it is what `COMMANDS.md` assumes throughout.
4. **The index is a binary file** (`.git/index`), not a JSON/text
   convenience file. Its exact format is in `ARCHITECTURE.md §3`.
5. **Refs are just files** containing a 40-char SHA (or, for symbolic refs,
   `ref: refs/heads/<name>\n`).
6. **Nothing is mutable.** Commits, trees, and blobs are never edited in
   place. "Changing history" (rebase, amend, reset) always creates new
   objects and moves a ref/pointer; old objects are simply left
   unreferenced until garbage collected.

## 2. Recommended implementation language

Any systems/general-purpose language with: raw byte I/O, a zlib binding,
and a SHA-1 implementation. Reference implementation is C; Go and Rust are
good ergonomic choices (`flate2`/`libz-sys` + `sha1` crates, or Go's
`compress/zlib` + `crypto/sha1`). Avoid languages without easy byte-level
control (e.g. don't build the object layer on top of a JSON-first runtime).

## 3. Source tree layout for `mygit` itself

```
mygit/
├── PACKAGE.md              (this file — ship it in the repo as design doc)
├── ARCHITECTURE.md
├── COMMANDS.md
├── PROTOCOL.md
├── src/
│   ├── main.[ext]          # CLI entrypoint, arg dispatch table (§5)
│   ├── objects/
│   │   ├── mod             # Object trait/interface: Blob, Tree, Commit, Tag
│   │   ├── blob
│   │   ├── tree
│   │   ├── commit
│   │   └── tag
│   ├── odb/                # object database: read/write loose + packed
│   │   ├── loose
│   │   └── pack
│   ├── index/               # staging area binary format, entry diffing
│   ├── refs/                # ref resolution, packed-refs, symbolic refs
│   ├── config/              # INI-style config parser (.git/config, ~/.gitconfig)
│   ├── ignore/               # .gitignore pattern matcher
│   ├── diff/                 # Myers diff algorithm, patch formatting
│   ├── merge/                 # three-way merge, conflict marker generation
│   ├── pack/                  # packfile writer/reader, delta compression
│   ├── transport/             # local, smart-http, ssh transports (§ PROTOCOL.md)
│   ├── commands/               # one module per porcelain/plumbing command
│   │   ├── init.*, add.*, commit.*, status.*, log.*, diff.*, branch.*,
│   │   ├── checkout.*, switch.*, merge.*, rebase.*, reset.*, revert.*,
│   │   ├── cherry_pick.*, stash.*, tag.*, remote.*, fetch.*, pull.*,
│   │   ├── push.*, clone.*, show.*, blame.*, reflog.*, gc.*, fsck.*,
│   │   └── plumbing/ (hash_object.*, cat_file.*, write_tree.*, ...)
│   └── cli/                  # argument parsing, help text, exit codes
└── tests/
    ├── unit/                 # per-module tests
    └── integration/          # golden-file tests: run mygit and real git
                               # side by side on the same operations, diff
                               # the resulting .git directory tree + stdout
```

## 4. The `.git` directory the tool must produce

```
.git/
├── HEAD                 # "ref: refs/heads/main\n"
├── config               # INI format, see ARCHITECTURE.md §5
├── description           # unused by porcelain, cosmetic
├── index                 # binary staging area, ARCHITECTURE.md §3
├── objects/
│   ├── ab/cdef...        # loose objects: 2-char dir + 38-char filename
│   ├── pack/              # *.pack + *.idx files
│   └── info/
├── refs/
│   ├── heads/<branch>     # 40-char SHA, one file per local branch
│   ├── tags/<tag>
│   └── remotes/<remote>/<branch>
├── packed-refs            # flat file, fallback/compaction for refs/
├── logs/                  # reflogs
│   ├── HEAD
│   └── refs/heads/<branch>
├── hooks/                 # sample shell scripts, not required for v1
├── info/
│   └── exclude            # repo-local gitignore, not versioned
└── COMMIT_EDITMSG          # scratch file used by `commit` for the editor
```

Every field, byte offset, and encoding referenced above is fully specified
in `ARCHITECTURE.md`. Do not guess at formats — an agent that improvises
the index binary layout will produce a `mygit` that real Git cannot read,
which fails the "byte-compatible" requirement in §0.

## 5. CLI dispatch shape

```
mygit <command> [--flag ...] [args ...]
```

- Unknown command → print list of common commands (mimic `git help`),
  exit code 1.
- `mygit <cmd> -h` / `--help` → per-command usage, exit 0.
- Global flags before the subcommand (`mygit -C <path> status`,
  `mygit --git-dir=<path> ...`) must be parsed and applied before
  dispatching.
- Exit codes matter: `0` success, `1` generic failure (e.g. merge conflict,
  nothing to commit), `128` fatal/usage error (not a repo, bad ref, bad
  object) — real Git scripts rely on this distinction; replicate it.

Full command list, per-command flags, and expected stdout/stderr shapes
are in `COMMANDS.md`.

## 6. Build order (do not build features out of this order)

Later phases depend on earlier ones being correct, and integration tests
compare against real `git` at every phase boundary — build phase N,
validate against real git, only then proceed to phase N+1.

1. **Object layer**: `hash-object`, `cat-file` — prove blob read/write and
   SHA-1/zlib correctness against real git's own object files.
2. **Index**: `update-index`, `ls-files`, then `add`/`status` — prove the
   binary index format round-trips with real git's index.
3. **Tree/commit plumbing**: `write-tree`, `commit-tree`, then `commit`
   porcelain, `log`.
4. **Refs & branches**: `symbolic-ref`, `update-ref`, `branch`,
   `checkout`/`switch`, `HEAD` detachment.
5. **Diff & merge**: `diff`, `merge` (three-way + conflict markers),
   `rebase` (implement as: sequence of cherry-picks onto a new base).
6. **History rewriting**: `reset` (soft/mixed/hard), `revert`,
   `cherry-pick`, `reflog`.
7. **Packfiles & gc**: `pack-objects`, `unpack-objects`, `gc`, `fsck`.
8. **Networking**: `clone`, `fetch`, `pull`, `push` over local + smart-HTTP
   transports (see `PROTOCOL.md`). This is the hardest phase; do it last.
9. **Ergonomics**: `stash`, `blame`, `tag` (lightweight + annotated),
   `.gitignore` matching, `config` command, `remote` management.

## 7. Testing strategy

For every command, the agent should generate a golden-file test that:
1. Creates a scratch directory, runs a sequence of operations through
   **real** `git`, and snapshots `.git/` + stdout.
2. Runs the identical sequence through `mygit` on a fresh scratch dir.
3. Diffs the two `.git/` trees structurally (objects present, refs equal,
   index entries equal — timestamps in the index may legitimately differ,
   everything else must not) and diffs stdout for porcelain commands with
   `--porcelain`/plumbing commands (script-stable output).

This is the acceptance bar: **if `mygit clone` of a `mygit`-authored repo
via real `git clone` succeeds, and vice versa, the implementation is
correct enough to ship.**

## 8. What is explicitly out of scope for v1

Submodules, worktrees, sparse-checkout, partial clone / shallow fetch
edge cases, GPG-signed commits/tags, `git-lfs`, rerere, bisect automation
beyond basic `bisect start/good/bad`. Note these as stubs that print
"not implemented" with exit code 1 rather than silently no-op — a script
depending on real git behavior should fail loudly, not corrupt state.
