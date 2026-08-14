# H26 Watchface File Format Specification (reverse engineering)

> Technical reference document for the binary format of H26 watchfaces (Xiaomi Vela / Mi Band ecosystem). Transcribed from the reverse-engineering worksheet and reorganized into a readable spec / technical prompt.

## 1. Overall file structure

The file consists of, in order:

1. **Header**
2. **Graphical blocks** (image/preview blocks)
3. **Block with internal addressing** (offset so far only observed right after the preview)
4. **UI Table**

## 2. Header

Byte layout, hex offsets:

```
Offset  00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
000000  53 62 40 2A 4F 32 47 47 00 0B 50 43 00 00 00 20   <- wf name / preview offset
000010  00 00 2A 7F 00 00 2A 9F 00 00 00 00 00 00 7B 42   <- preview length
```

Known fields:
- **wf name**: watchface name (string, presumably in the trailing block at 0x17).
- **preview offset**: pointer to the preview block.
- **preview length**: length of the preview block.
- **offset of the block with internal addressing**: not yet reliably located; observed immediately after the preview.
- **length of the block with internal addressing**: not yet determined.
- **UI table offset**: not yet determined.

## 3. Graphical blocks

Each image block has a **Header** and a **Data** section. Observed block types (block type tag):

### 3.1 `32 bit BGRA paletted (256 colors)`
```
Header: 4B 01 FC 27 03 F6 A1 19 6F 2A 00 00 00 00 00 00   -> unpacked data length
Data:   LZ4 stream                                         -> block length (byte count not certain)
                                                             -> combined width and height
```

### 3.2 `BGR565 raw`
```
Header: 49 01 00 0E 01 80 40 0B EB 06 00 00 00 00 00 00
Data:   LZ4 stream
```
Width/height example:

| bytes | value | width | height |
|---|---|---|---|
| F6 A1 19 | 19 A1 F6 | 19A | 1F6 |

### 3.3 `BGR565A raw`
```
Header: 48 01 00 0E 01 80 40 0B EB 06 00 00 00 00 00 00
Data:   LZ4 stream
```

### 3.4 `Common JPG image`
```
Header: 09 00 C5 3A 00 F6 A1 19 00 00 00 00 00 00 00 00
Data:   raw jpg image
```

### 3.5 `Common GIF image`
```
Header: 03 00 F0 CC 13 C0 01 17 00 00 00 00 00 00 00 00
Data:   raw gif image
```

## 4. UI Table

> "UI Table is just a sequential UIItems" — the UI Table is a sequence of `UIItem`s.

### 4.1 Common `UIItem` header

Generally **5 fields of 4 bytes each** (5×4 bytes):

| Field | Size |
|---|---|
| Type | 4b |
| Sub Type | 4b |
| Align | 4b |
| X | 4b |
| Y | 4b |

Note: at least X and Y are **signed** and can be negative (probably all 4-byte values in the UI Table are signed).

### 4.2 Type 0 — Layout

Header (5×4 bytes): Type, Sub Type, Align?, X, Y.

**Sub Type** (at the "extended bytes" offset):
- `00 00 00 8C` → **regular** watchface layout
- `00 00 00 8D` → **AOD** (Always-On Display) watchface layout

**Extended bytes** of the layout:

| Layouts counter | UIItems counter | UIItem Indexes (regular) | UIItems counter | UIItem Indexes (AOD) |
|---|---|---|---|---|
| 4b | 4b | 4b × N | 4b | 4b × N |

Each "UIItem Indexes" is an array of indices (4 bytes each) pointing to the UIItems that make up that layout.

There is also a second extended-bytes block with signature `00 00 00 34`, with two variables not yet understood (always observed = 0) — "unknown / unknown" (4b + 4b).

### 4.3 "Pack of frames" (recurring generic structure)

Used by several types (e.g. Type 1-3/5/6/18/56, Type 37, etc.) to reference multiple frames/animations:

```
extended bytes: Frame counter | Frame offset | Frame length | UIItem indexes | Frame offset | Frame length | UIItem indexes | ...
                4b             4b             4b             4b (xN)          4b             4b             4b (xN)
```

### 4.4 Type 1-3, 5, 6, 18, 56 — Font

Used as fonts; as far as observed they only contain **internal block offsets**. Generic pack-of-frames structure (see 4.3).

### 4.5 Type F — Hands

Standard header (5×4 bytes: Type, Sub Type, Align, X, Y).

**Sub Type**:
- `00 00 00 0B` → **Hour hand**
- `00 00 00 0C` → **Minute hand**
- `00 00 00 0D` → **Second hand**

Note: `rX, rY` is a pointer on the frame (from the frame's top-left corner) = **rotation point**.

**Extended bytes**:
```
Frame counter | rX | rY | Frame offset | Frame length | UIItem indexes | rX | rY | Frame offset | Frame length | UIItem indexes
4b              4b   4b   4b             4b              4b (xN)          4b   4b   4b             4b              4b (xN)
```

### 4.6 Type 14 — Animation

Reduced header (3×4 bytes): Type, Sub Type, Align?.

**Sub Type**:
- `00 00 00 34` → animation
- `00 00 00 3B` → animation

**Extended bytes**:
```
unknown | X | Y | Frame counter | Frame offset | Frame length | UIItem indexes | Frame offset | Frame length | UIItem indexes
4b        4b  4b  4b               4b             4b             4b (xN)          4b             4b             4b (xN)
```

There is also a variant with signature `00 00 00 70` that adds an extra layer:
```
unknown | unknown | X | Y | Frame counter | Frame offset | Frame length | Frame offset | Frame length
4b        4b        4b  4b  4b               4b             4b             4b             4b
```

### 4.7 Other types — generic "pack of frames"

Minimal structure:
```
extended bytes: X | Y | Frame counter | Frame offset | Frame length | UIItem indexes | Frame offset | Frame length | UIItem indexes
                4b  4b  4b               4b             4b             4b (xN)          4b             4b             4b (xN)
```

### 4.8 Type 37 — Button (?) / reference to system screen

Header (5×4 bytes): Type, Sub Type, Align?, X, Y.

Content: "**button?**".

**Extended bytes**:
```
unknown | Width | Height | 30 bytes of strings (0-terminated)
4b        4b       4b       ...
```

Value `always = 3` observed in one field, followed by a list of known system screen strings/names:
- `WeatherScreen`
- `CompassScreen`
- `StepDetailScreen`
- `HRScreen`

### 4.9 Type 47, 48, 4B, 4C — Angled fonts

Header (5×4 bytes): Type, Sub Type, Align, X, Y.

Note: this is **not an image rotation**, but a position shift applied to each successive character.

**Extended bytes**:
```
Counter | dX? | dY? | Frame offset | Frame length | UIItem indexes | Frame offset | Frame length | UIItem indexes
4b        4b    4b    4b             4b             4b (xN)          4b             4b             4b (xN)
```

Note: the `Counter` also counts dX and dY, i.e. it equals `frame count + 2`.

### 4.10 Type 5B — Solid rectangle

Header (5×4 bytes): Type, Sub Type, Align, X, Y.

Content: "**solid rectangle?**".

**Extended bytes**:
```
Counter? | Width? | Height? | color (B,G,R) | (alpha not used)
4b         4b        4b        4b (count=3, then 3 bytes B/G/R)
```

## 5. General notes / open uncertainties

- It is not yet clear where in the header the offset and length of the "block with internal addressing" point exactly.
- The UI Table offset within the main header is not yet determined.
- Several fields are marked with `?` in the original material (Align?, Width?, Height?, Counter?, dX?, dY?, rX/rY), indicating the interpretation is probable but not 100% confirmed.
- All integer values in the UI Table appear to be 4-byte, signed, big-endian (based on the hex examples provided).

---
*Document generated from the analysis file `h26.xlsx` (sheets "File structure", "Header", "Graphical blocks", "UITable" + hex example sheets).*
