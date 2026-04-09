# Album Art on the HiBy R3 Pro II

This document covers album art behavior, optimal sizing, and tools for preparing art for the best performance on the R3 Pro II.

## Display Specifications

| Property | Value |
|----------|-------|
| Framebuffer resolution | 480×720px |
| Album art display area | 360×360px |
| Recommended art size | 360×360px |
| Recommended format | JPEG |

The device display area for album art is 360×360px. Any image larger than this is downscaled by the device at runtime, which wastes CPU time and RAM on every display. Scaling art to 360×360px before loading it onto the SD card eliminates this overhead entirely.

## Why Size Matters

The R3 Pro II uses an Ingenic X1600 single core MIPS processor with 56MB total RAM (~12MB available while running). Loading and scaling large album art images is expensive on this hardware:

- Hi-res covers (1000–1800px) take noticeably longer to display during fast track skipping
- The device must decode, scale, and render each image in software
- Larger images consume more of the limited available RAM
- The image cache feature (`tf_image_cache_enable` in `config.json`) was tested and found to cause audio lag after ~5 songs due to the CPU cost of converting images to 480p RGBA — it is disabled by default and should remain so

At 360×360px, album art loads in under half a second even for rapid track changes under normal listening conditions.

## Resizing Album Art with ffmpeg

To resize embedded album art in audio files and save it as an external cover image:

```bash
ffmpeg -i "input.flac" -an -vf scale=360:360 cover.jpg
```

To batch resize all embedded art across a music library:

```bash
find /path/to/music -name "*.flac" -o -name "*.mp3" | while read f; do
    dir=$(dirname "$f")
    if [ ! -f "$dir/cover.jpg" ]; then
        ffmpeg -i "$f" -an -vf scale=360:360 "$dir/cover.jpg" 2>/dev/null
    fi
done
```

To resize an existing image file directly:

```bash
ffmpeg -i cover_original.jpg -vf scale=360:360 cover.jpg
```

## External vs Embedded Art

The R3 Pro II supports both embedded album art (stored inside the audio file) and external cover art (stored as a separate image file in the same folder as the audio files).

The PC Database Updater script (`tools/Update_Database.py`) looks for external cover art using these prioritized filenames:

- `cover.jpg` / `cover.jpeg` / `cover.png`
- `folder.jpg` / `folder.jpeg` / `folder.png`
- `front.jpg` / `front.jpeg` / `front.png`
- `albumart.jpg` / `albumart.jpeg` / `albumart.png`

Results are cached per directory, so only one image per album folder is used.

## Recommended Workflow

1. Prepare your music library with external `cover.jpg` files at 360×360px per album folder
2. Run `tools/Update_Database.py` to generate the music library database
3. The database will reference the correctly sized cover art
4. Copy the database to the device via **Settings → Database Manager → Copy Database from SD**

This gives the fastest possible album art display with no runtime scaling overhead.

## Screenshot Capture and Album Art

Note that album art on the now playing screen is rendered via the Ingenic IPU (Image Processing Unit) which composites directly to the display hardware, bypassing the `/dev/fb0` framebuffer. Album art thumbnails in list views are also IPU-rendered. This means album art will not capture correctly via the ADB framebuffer screenshot method — see `docs/SCREENSHOTS.md` for details.
