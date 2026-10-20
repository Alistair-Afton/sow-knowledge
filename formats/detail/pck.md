# `.pck` — AKPK v1 (Wwise file package)

> **Status:** **proven** — all 245,134 media entries extracted
> **Provenance:** all 245,134 media entries extracted from real `.pck` archives.

Sound packages under `sound/<lang>/` (`external*.pck`, `vo*.pck`) and
`sound/streaming*.pck`. 77 files carrying ~245k embedded `.wem` streams
(`external.pck` alone = 662 MiB / 25,449 entries; `streaming.pck` ≈
967 MiB). This is the standard Wwise "file package" container.

## Layout

```text
+0x00  char[4] "AKPK"
+0x04  u32     header size − 8  (index-section end − 8)
+0x08  u32     version = 1
+0x0c  u32     sizeof(Languages)
+0x10  u32     sizeof(BanksLUT)
+0x14  u32     sizeof(STM)
+0x18  u32     sizeof(FileIndex)
+0x1c  sections in order:
       Languages  {u32 n, n×{u32 str_off, u32 lang_id}, UTF-16LE pool}
                  str_off relative to section start; names NUL(2B)-
                  terminated. Corpus names: "sfx" (non-localized packs),
                  localized packs carry the voice language.
       BanksLUT   {u32 n, n×20B} — n = 0 in all 77 corpus files
       STM        {u32 n, n×{u32 id, u32 unk, u32 size, u32 off, u32 unk}}
       FileIndex  {u32 n, n×{u64 id, u32 unk, u32 size, u64 off}}
data   media blocks
```

Every indexed `off` lands exactly on a `RIFF` (`.wem`) header and
`size` equals the stream's own RIFF length — verified on
245,134/245,134 entries (27,250 STM + 217,884 file).

`Pck::stm_media`/`Pck::file_media` slice media directly out of the
package buffer (no copy).

## Open

- STM/FileIndex `unk` fields (likely language-id + padding flags).
- Bank-level LUT entries (corpus ships zero).
