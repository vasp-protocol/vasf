# VASF — Visual Automation State Format

Version: 1.0-draft | Status: Open RFC | Published: May 2026  
Reference implementation: [farscry](https://github.com/teles-forge/farscry)

## What VASF Is

A binary container format for sequences of VASP visual states.  
Designed for recording, replaying, and analyzing computer-use agent sessions.

Like VASP standardizes one visual frame, VASF standardizes a session of frames.

## Design Principles

1. **Incremental write** — valid at any point if process dies mid-session
2. **Deduplication by pHash** — only unique visual states are stored
3. **Semantic content only** — no raw pixels in the format
4. **Self-contained** — one file contains the complete session

## Binary Format

### Header (18 bytes)

| Offset | Size | Type   | Description                    |
|--------|------|--------|-------------------------------|
| 0      | 4    | bytes  | Magic: ASCII `VASF`           |
| 4      | 2    | u16 LE | Format version (currently 1)  |
| 6      | 4    | u32 LE | Number of frames               |
| 10     | 8    | i64 LE | Created at (unix seconds)     |

### Frame (variable length)

| Offset | Size | Type   | Description                         |
|--------|------|--------|-------------------------------------|
| 0      | 8    | bytes  | State ID (raw pHash bits)           |
| 8      | 8    | i64 LE | Timestamp (unix milliseconds)       |
| 16     | 4    | u32 LE | VASP data length                    |
| 20     | N    | bytes  | VASP text (zstd compressed)         |
| 20+N   | 4    | u32 LE | Delta data length (0 = no delta)    |
| 24+N   | M    | bytes  | Delta text (zstd compressed, if >0) |

### State ID Algorithm

Same as VASP `state_id`:

1. Resize image to 32x32 (nearest-neighbor)
2. Convert to grayscale: `0.299R + 0.587G + 0.114B`
3. Compute 32x32 2D DCT-II
4. Extract top-left 8x8 coefficients (64 values)
5. Exclude `DCT[0][0]` (DC component)
6. Compute mean of remaining 63 values
7. `bit=1` if value > mean, else `0`
8. Pack 64 bits → `[u8; 8]`

### Deduplication

Frames are deduplicated by pHash Hamming distance.  
Default threshold: **10 bits**.  
Frames with Hamming distance ≤ threshold vs previous frame are skipped.  
Only unique visual states are stored.

## Session Statistics

A VASF reader can compute:

```
total_frames:    count of all frames seen (including duplicates)
unique_states:   count of frames stored
dedup_rate:      (total - unique) / total
token_reduction: (total * 2765) / (unique * 200)
```

where `2765 = 1920*1080/750` (vision tokens at 1080p) and `200` = average VASP output tokens.

## Reference Implementation

```sh
farscry pack   frames/         -o session.vasf
farscry timeline               session.vasf
farscry info                   session.vasf
farscry serve  --mcp --record  session.vasf
```

Source: [github.com/teles-forge/farscry](https://github.com/teles-forge/farscry)

## Status

Open RFC. Implementation exists.  
Comments welcome: [github.com/vasp-protocol/vasf/issues](https://github.com/vasp-protocol/vasf/issues)

## License

Apache 2.0
