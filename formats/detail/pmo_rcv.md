# `.pmo` + `.rcv` — motion/behavior curves

> **Status:** **verified** — motion-curve containers decoded
> **Provenance:** motion-curve containers decoded on the real corpus.

Per-character motion data under `anim/` and `behaviors/` (all
`player_*`).

## `.pmo` — PARA v1 (24 files)

Parametric-transition tables (`player_el2_idle_to_walkstl_l_major`…):

```text
+0x00  char[4] "PARA"
+0x04  u32     version = 1
+0x08  u32     channels (2..9)
+0x0c  u32     frames (500 everywhere)
+0x10  u32     block count (1 or 2)
+0x14  f32×3   params
+0x20  frames × (4·channels + 8)B f32 records
       + optional second block (~frames×4B index table + 12B) when
         blocks == 2
```

`len = 0x20 + frames × (4ch+8)` exactly on all 21 single-block files.

## `.rcv` — RCVA v2 (45 files)

Recovery/height curves for mantle/vault behaviors:

```text
+0x00  char[4] "RCVA"
+0x04  u32     version = 2
+0x08  u32     block count
+0x0c  u32     0
+0x10  u32     field (correlates with body size)
+0x14  i32×2   bounds pair
+0x1c  blocks: {u32 n, u32 =2, u32, u32} + n-sample f32 data
```

All block headers in a file share one `n` — the value at `+0x1c`
(4..47 observed). 30/45 files carry this layout; the other 15 have a
denser variant header (no markers) and expose `blocks = []`.

`Pmo`/`Rcv` in `anduril-anim`; records stay raw f32 bytes — the
semantic channel mapping is TBD (engine-side names are hashed).
