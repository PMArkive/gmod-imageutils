# gmod-imageutils

Client-side image toolkit for Garry's Mod. Two modules, both in `lua/includes/modules/`:

- `imgparse` — zero-decode binary parsers that pull metadata (dimensions, format, etc.) straight from PNG/JPG/VTF headers.
- `urlimage` — async HTTP image fetching with an on-disk LRU cache backed by SQLite, exposing `surface.URLImage` / `render.URLMaterial` that resolve to a `Material` once downloaded.

**NOTE**: Readme is ai-generated, but mostly correct.

### Minimal example

```lua
require("urlimage")

local img = surface.LazyURLImage("https://placehold.co/300x256.png",'smooth')

hook.Add("HUDPaint", "test", function()
    local w,h = img()
    if not w then return end
    surface.SetDrawColor(255,255,255,255)
    surface.DrawTexturedRect(0, 0, w, h)
end)
```

## Dependencies

- [`gmod-sqliteext`](https://github.com/Metastruct/sqliteext) — required for the persistent SQLite cache. If `sql.obj` is missing, `urlimage` degrades to an in-memory cache (no persistence, no purge) rather than failing.

## Using in your addon

Presently embedding this as standalone is not possible. There is also not yet a shared workshop addon. Maybe there should be.

## imgparse

Header-only parsers — no image decoding, safe on arbitrary/oversized files. They take an open binary `File` handle (PNG also accepts a path), and return the parsed header table.

| Function | Returns |
|---|---|
| `file.ParsePNG(fh \| path)` | `{ width, height, bit_depth, color_type, compression_method, filter_method, interlace_method, rendering_intent }` — `error()` on invalid data |
| `file.ParseJPG(fh)` | `{ width, height }` — skips segments until the first SOF0/SOF2 (`FFC0`/`FFC2`) marker |
| `file.ParseVTF(fh)` | `{ width, height, version = {v1, v2}, headerSize }` or `nil, err` — validates `"VTF\0"` magic, version and power-of-two dimensions |

`file.ParsePNG` walks the chunk stream (length/type/CRC framing), parses IHDR, sRGB, gAMA, cHRM, skips/validates IDAT (zlib header + Adler-32 check only, no decompression), and stops at IEND.

Magic-byte sniffers (used by `urlimage` for format detection):

- `string.IsPNG(bytes)` — `\137PNG\r\n\032\n`
- `string.IsJPG(bytes)` — `\255\216` (FFD8)
- `string.IsVTF(bytes)` — `VTF\0`

```lua
require("imgparse")

local fh = file.Open("data/cache/uimg/42.png", "rb", "DATA")
local png = file.ParsePNG(fh)          -- { width, height, color_type = 6, ... }
fh:Close()

print(string.IsPNG(file.Read("foo.vtf", "DATA")))  -- false
```

## urlimage

Download → sniff format → validate dimensions → cache to disk → yield a `Material`, all asynchronously.

### Public API

| Function | Notes |
|---|---|
| `surface.URLImage(url, data?)` | returns a [thunk](https://en.wikipedia.org/wiki/Thunk); call it to poll readiness / draw |
| `surface.LazyURLImage(url, data?)` | same thunk contract, but the HTTP fetch only starts on the **first** call of the thunk |
| `render.URLMaterial(url, data?)` | 3D variant; calls `render.SetMaterial` instead of `surface.SetMaterial` |
| `urlimage.GetURLImage(url, data, isSurface)` | low-level: `mat, w, h` when ready, `false` while processing, `nil, err` on failure |
| `urlimage.GetFastDL()` | `sv_downloadurl` base used to prefix relative URLs; override via `urlimage.fastdl_override` |
| `urlimage.get_cache_info()` | `{ count, bytes }` from the DB |
| `urlimage.setDebug(bool)` / `urlimage.GetDebug()` | toggle `[UrlImg]` console logging |

### Thunk contract

Calling the returned thunk yields:

- `w, h, mat` — ready; `surface.SetMaterial`/`render.SetMaterial` has **already** been called for you
- `nil` — still processing, or permanently failed (the thunk becomes a no-op on failure)

Once the image resolves the thunk re-binds to the finished material, so repeated calls are cheap. Poll it until `w` is truthy.

**Do not call `surface.URLImage`/`render.URLMaterial` every frame** — the module counts consecutive per-frame calls and `ErrorNoHalt`s (or `error`s in the forks) after ~15–18 frames. Hold the returned thunk; silence the check with global `IKNOWWHATIMDOING = true`.

`data` is forwarded verbatim to `Material(path, data)` for PNG/JPG (e.g. `'smooth'` for bilinear-filtered UI icons, `'unlitgeneric'`). Ignored for VTF, which always builds via `CreateMaterial` — `UnlitGeneric` for surface, `VertexLitGeneric` for render — with `$vertexcolor`/`$vertexalpha` and an `AnimatedTexture` proxy.


### URL handling (outdated)

`FixupURL` normalizes inputs before fetching:

- relative paths get prefixed with `GetFastDL()` (`sv_downloadurl`), so `materials/foo.png` → `<fastdl>/materials/foo.png`
- OneDrive `redir?` → `onedrive.live.com/download?`
- GitHub `.../blob/...` → `.../raw/...`
- `www.dropbox.com/s/...?dl=0/1` and `dl.dropboxusercontent.com` → direct download URL

### Fetch pipeline

1. Check the in-memory `cache[url]` (also dedupes concurrent in-flight requests for the same URL), then the SQLite record.
2. Valid cached record → material immediately; invalid record → row deleted and re-fetched.
3. New URL: `HTTP{ method = HEAD }` for an early `Content-Length` check, then `http.Fetch`.
4. Body validated: `8 < len <= 25 MB`, magic-sniffed via `string.IsPNG/JPG/VTF`, written to `data/cache/uimg/<fileid>.<ext>`, dimensions parsed via `imgparse`, record stored.

### Limits

- body: `8 < len <= 25 MB` (4*2048*2048+1KB ≈ 16 MB in the `_fsize` fork); HEAD `Content-Length > 15 MB` rejected before the download starts
- dimensions: clamped to `min(render.MaxTextureWidth(), 2048)`; larger → `excessive dimensions`, record dropped
- unrecognized magic → `unknown format`
- HTTP status ≠ 200 → failure, record rolled back

### Cache & LRU

- files: `data/cache/uimg/<fileid>.<ext>`
- metadata: SQLite table `urlimage` (via `sql.obj`), created+migrated (`ADD COLUMN file_size`) on load:

```
url       TEXT UNIQUE CHECK(url <> '')
ext       TEXT CHECK(ext IN ('vtf','png','jpg'))
last_used INTEGER   -- relative epoch, 0 == unix 1477777422
fetched   INTEGER
locked    BOOLEAN   -- 1 = in use, exempt from purge
size      INTEGER   -- legacy column
w, h      INTEGER
fileid    INTEGER PRIMARY KEY AUTOINCREMENT
```

- every hit bumps `last_used` and sets `locked = 1`
- on load, all `locked` rows are cleared, then `do_purge` evicts unlocked rows `ORDER BY last_used ASC`, trimming to `MAX_ENTRIES = 2048`, deleting both the DB row and every file variant (`.vmt`, `.jpg`, `.png`, `.vtf`) for evicted fileids
- without SQLite (`sql.obj` nil) it still works: records live in memory and fileids come from a global counter (`URLIMAGE_EMERGENCY_UID`), so the cache is simply non-persistent

