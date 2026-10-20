# `.embb` — embedded section bundle (v5)

> **Status:** **partial**** — section layout decoded; consumers open
> **Provenance:** embedded-section layout decoded on the real corpus.

Sibling of each section's `.scripts` — e.g. `sotp_desert_06.embb` next to
`sotp_desert_06.scripts`. A container that embeds the section's own
asset set so a layer loads as one stream.

## Header (verified on sotp_desert_06)

```text
+0x00  char[4] "EMBB"
+0x04  u32     version = 5
+0x08  u32     file-list byte size (NUL-separated member paths)
+0x0c  u32     file count
+0x10  char[..] file list — member path strings
```

Followed by a per-member directory (17 x u32 run observed, ~0x40
strides — member record offsets, layout not yet decoded) then member
blobs. Members observed in sotp_desert_06: the section's own
`.wldclnt` (embedded copy, stub string pool), `.scripts` (SCRI),
`.hkai`, `.objres`, `.uis`, `.uiwd`, `.veggrid`, `.terrain`,
`.inst`, plus shared fx/mesh/skel assets.

Member layout is fully decoded (matches Project Nemesis'
`sow/embb.hexpat`): after the file list come `fileCount` u32 offsets
*into the path array*, then members contiguously as
`{u32 size, u8 data[size]}`.

## Section object table — inside the `.objres` member

The embedded `*.objres` member carries an `OBJL` chunk (ver 5) whose
name table is the section's live object database — see
`objres.md`:

- `Statu_00.R.QR`, `Statu_00.SO`, `Statu_00.TP`, `Statu_00.R.*` —
  script-object instances by class suffix (QR/SO/TP/Utility/...)
- `SP06_Spwn_*` — named spawners, `SP06_TP_Captain_*` — teleport
  points (positions presumably alongside in the member's records)
- `GTeleportPoints`, `GOrcs`, `GNodes` — **named groups**, exactly
  what scripts bind via `Group:Create(scriptId, "GTeleportPoints")`
- `SP06_QuestStartRelay`, `MR_00`, etc.

Decoding this member (name -> transform -> group membership) is what
makes `Group:Create(name)` return a real collection: `Size`, `At`,
`Remove`, `Insert`, and `TeleportPoint.Cast`/`GetDistanceToObject`
answer real data — the last corpus failure (`sotp_desert_06`
re-roll loop) needs exactly that.

CLI: `embb <section.embb> objects` extracts the embedded `.objres`
member and lists its OBJL names.
