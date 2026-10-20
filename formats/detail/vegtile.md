# `.vegtile` — `VEGT` v3 vegetation tiles

> **Status:** **verified**** — vegetation cells decoded
> **Provenance:** vegetation cells decoded on the real corpus.

One tile per streamed vegetation cell under `worlds/**/<world>_<n>v.vegtile`
(927 files in this install).

## Layout (verified — records end exactly at EOF on all files)

```text
+0x00  char[4] "VEGT"
+0x04  u32     version = 3
+0x08  f32×3   AABB min corner (world coordinates)
+0x14  f32×3   AABB max corner (or size — a sparse benchmark tile shows 121.07³)
+0x20  u32     species_count
+0x24  species_count × record:
       +0x00  u16      name_len
       +0x02  char     name[name_len]     (not NUL-terminated)
       +...   u32      instance_count
              count × 16B packed instance
```

`anduril-veg` decodes the container:

```text
mtnpass_109v.vegtile: species=5 instances=763
  aabb = [-5429.8, 40460.6, 47721.2] - [3111.8, 530.0, 4224.0]
```

Across the corpus: **927/927 parse exactly, ~1,708,046 instances**,
120 distinct species names (`MtPGrTussGreen01`, `MtPShrubDead01`,
`MtPsWlkOvRck`, `BeechAdult/Teen/Old`, `DES_Palm01/Eucalyptus01`,
`AloeVera_Adult`, `DesGrassGreen01`, …).

## 16-byte instance (opaque in v1)

Packed per-placement data. Bytes like `75 24 53 18 e9 24 5c ff` +
recurring `7f 7f 00` runs suggest quantized position + rotation/normal
+ scale — consistent with instanced-vegetation draws. Decoding the
fields (likely quantized vs. the tile AABB) is future work.

## Known unknowns

- Inner decode of the 16B instance (position/rotation/scale packing).
- Species-name → model/material binding (a `veg_<name>.mesh`/`.mat`
  convention, or a gamedb species table — unverified).
- Exact AABB semantics (min+size vs min+max — both plausible on the
  samples checked).
