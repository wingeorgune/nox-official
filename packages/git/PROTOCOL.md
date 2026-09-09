# PROTOCOL.md — Transports, negotiation, packfiles

Referenced from `COMMANDS.md` (clone/fetch/pull/push) and
`ARCHITECTURE.md §2` (packed objects). Implement this last, per
`PACKAGE.md §6` phase 8.

## 1. Transports

Support three, in this priority order (easiest → hardest):

1. **Local (`file://` or bare path)**: "transport" is just direct
   filesystem access to the remote's `.git` directory — read its refs
   and objects directly, no protocol needed. Build and validate the
   negotiation/pack logic below against this transport first.
2. **Smart HTTP(S)**: two endpoints —
   `GET <url>/info/refs?service=git-upload-pack` (fetch) or
   `git-receive-pack` (push) for ref advertisement, then
   `POST <url>/git-upload-pack` / `git-receive-pack` with the
   negotiation request body, response is a packfile stream.
   Requires implementing the **pkt-line** framing format (§2) and,
   for private repos, HTTP Basic Auth or bearer token from a credential
   store/env var.
3. **SSH**: shell out to a system `ssh` binary invoking the remote's
   `git-upload-pack '<path>'` / `git-receive-pack '<path>'` executable
   over the SSH session's stdin/stdout, then speak the same pkt-line
   protocol as HTTP over that pipe. Don't reimplement SSH — delegate to
   the OS's ssh client as real Git does.

## 2. pkt-line framing

The wire format for both refs and negotiation is a sequence of packets:

```
4 hex-digit ASCII length (includes these 4 bytes) + payload
"0000" = flush-pkt (end of a section)
"0001" = delim-pkt (protocol v2 only)
```

E.g. `0032want 74730d410fcb...\n` — length `0x0032` = 50 bytes total,
payload `want 74730d410fcb...\n`. Every negotiation line below is sent
as one pkt-line.

## 3. Ref advertisement (start of every fetch/push)

Server responds with one pkt-line per ref:

```
<oid> <refname>\0<capabilities>\n     (first line only, capabilities appended)
<oid> <refname>\n
...
0000
```

`<capabilities>` is a space-separated list the server supports
(`multi_ack_detailed`, `side-band-64k`, `ofs-delta`, `agent=...`, etc.)
— the client's subsequent negotiation lines should only use capabilities
present in this list. Implement at minimum: `ofs-delta` (offset-delta
packs, smaller), `side-band-64k` (multiplex pack data + progress +
error on one stream, §5).

## 4. Fetch negotiation (want/have)

Client wants some refs it doesn't have the full history for. Algorithm:

1. Client sends `want <oid>` for each ref tip it wants (its local
   `refs/remotes/<remote>/*` targets after this fetch), then `0000` flush.
2. Client sends `have <oid>` for oids it already has locally (walk its
   own ref tips and recent history — don't enumerate the whole object
   db, that defeats the purpose), interspersed with periodic flush
   packets so the server can respond incrementally with `ACK <oid>` for
   any common ancestor found (this lets negotiation stop early instead
   of walking full history every time).
3. Client sends `done`.
4. Server computes the minimal object set: everything reachable from the
   `want`s, minus everything reachable from acknowledged `have`s, and
   streams it back as a single packfile (§5).

For a first-time `clone` (no local history), skip `have` entirely — just
`want` every advertised ref tip and `done` immediately; the server sends
the full reachable object set.

## 5. Packfile format (also used for local `gc`, and as the fetch/push payload)

```
Header (12 bytes):
  4 bytes   "PACK"
  4 bytes   version (2, big-endian uint32)
  4 bytes   number of objects (big-endian uint32)

Then, for each object:
  Variable-length header encoding (type + size):
    First byte: bit 7 = continuation flag, bits 6-4 = type
                (1=commit, 2=tree, 3=blob, 4=tag, 6=ofs-delta, 7=ref-delta),
                bits 3-0 = low 4 bits of size.
    If continuation bit set, next byte(s): bit 7 = continuation,
                bits 6-0 = next 7 bits of size, little-endian chunk order.
  If type is ofs-delta: a variable-length negative byte offset (back to
    the base object earlier in this same pack) follows the header.
  If type is ref-delta: a raw 20-byte oid of the base object follows.
  Then: zlib-deflated content — either the full object content (types
    1-4) or delta instructions (types 6-7, format below).

Trailer (20 bytes): SHA-1 checksum of the entire preceding pack content.
```

**Delta instructions** (after inflating a delta object's content):
first two variable-length integers = base object size and result object
size (sanity checks), then a sequence of opcodes:
- `copy` (high bit of opcode byte set): read up to 4 offset bytes + up
  to 3 size bytes (each present only if its corresponding low bit in the
  opcode is set) → copy `size` bytes from `offset` in the **base**
  object into the output.
- `insert` (high bit clear): opcode byte's low 7 bits = a literal byte
  count N; the next N bytes (verbatim, not further encoded) are appended
  to the output directly.

**`.idx` file** (companion to `.pack`, not sent over the wire — the
client builds its own after receiving a pack): a fan-out table (256
entries, cumulative count of objects whose oid's first byte ≤ index)
for fast binary search, followed by sorted oid list, then a CRC32 per
object, then a packed-file-offset per object, then the pack's own
trailer checksum. Building this correctly is what makes `cat-file`/
`log`/etc. fast against packed objects instead of doing a linear scan.

**side-band-64k**: while streaming the packfile, the server prefixes
every chunk with a 1-byte channel indicator (`\x01` = pack data,
`\x02` = progress text for stderr, `\x03` = fatal error) so pack bytes
and human-readable progress can share one connection. Demultiplex on
receipt: channel 1 → feed to the pack parser, channel 2 → print to
stderr, channel 3 → print and abort.

## 6. Push

1. Client fetches ref advertisement (as in a normal fetch handshake,
   but talking to `git-receive-pack`).
2. Client sends update commands, one pkt-line per ref being pushed:
   ```
   <old-oid> <new-oid> <refname>\0<capabilities>\n   (first line)
   <old-oid> <new-oid> <refname>\n
   0000
   ```
   `<old-oid>` = what the client believes the remote ref currently is
   (from the advertisement) — this is the safety check; if the actual
   remote value has since changed, the server rejects with a
   non-fast-forward error (**check this server-side**, not just
   client-side, or the implementation has a race/security hole).
3. Client sends a packfile containing every object the remote is
   missing (walk local history reachable from the new oids, minus
   history reachable from the advertised remote refs — same
   reachability primitive as `merge-base`/`log` ranges).
4. Server applies the pack, updates each ref if its old-oid check
   passed, and responds (if `report-status` capability was negotiated)
   with per-ref `ok <refname>` / `ng <refname> <reason>` lines so the
   client can report exactly which refs succeeded.

## 7. Object walking primitive (shared by negotiation, push, and `log` ranges)

Implement one reusable function:

```
reachable(tips: [oid], exclude: [oid]) -> set[oid]
```

BFS/DFS from `tips` following commit `parent` links (and, when walking
for pack construction rather than just `log`, also descending into each
visited commit's `tree` and every tree's blob/subtree entries), stopping
a branch as soon as it hits something in `reachable(exclude, [])` or an
already-visited node. This single primitive backs: `merge-base`,
`log A..B`, fetch's "what don't I have", and push's "what does the
remote need from me." Build it once, well-tested, and call it everywhere
rather than re-deriving graph-walk logic per command.
