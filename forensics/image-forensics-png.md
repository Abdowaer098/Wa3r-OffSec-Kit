# Image Forensics: PNG

## Overview
PNG (Portable Network Graphics) uses lossless compression and supports transparency, making it distinct from JPEG in both structure and forensics approach. PNG's chunk-based format provides multiple hiding locations, and its lossless nature makes it ideal for LSB (Least Significant Bit) steganography.

## PNG File Structure

| Component | Hex signature | Description |
|-----------|--------------|-------------|
| PNG Header | `89 50 4E 47 0D 0A 1A 0A` | 8-byte magic number |
| IHDR chunk | | Image dimensions, color type, bit depth |
| IDAT chunk(s) | | Compressed image data |
| IEND chunk | `00 00 00 00 49 45 4E 44 AE 42 60 82` | End of PNG marker |

**Key insight**: Data appended **after the IEND chunk** is ignored by PNG renderers but may contain hidden content. This is the most common hiding spot.

## Analysis Workflow

### 1. Identify and validate
```bash
file image.png

hexdump -C image.png | head -5
# Should start with: 89 50 4e 47 0d 0a 1a 0a
```

### 2. Extract metadata
```bash
exiftool image.png
exiftool -a -u -g1 image.png
```

Check for `Comment`, `Description`, `Author`, and any custom text chunks (`tEXt`, `zTXt`, `iTXt`).

### 3. Check for data after IEND
```bash
# View end of file
hexdump -C image.png | tail -30

# Search for IEND position
grep -oba "IEND" image.png

# Check if file continues after IEND
binwalk image.png
```

If `binwalk` shows additional files after the IEND marker, extract them:
```bash
binwalk -e image.png
```

### 4. String analysis
```bash
strings image.png | grep -i "flag\|password\|secret\|key"
```

### 5. Steganography detection (LSB)
```bash
zsteg image.png           # Run all detection methods
zsteg -a image.png        # Aggressive mode

# Extract specific bit-plane data
zsteg -E "b1,rgb,lsb" image.png
zsteg -E "b1,r,lsb" image.png    # Red channel only
```

### 6. Chunk analysis
```bash
# List all PNG chunks
pngcheck -v image.png

# Detailed chunk inspection
identify -verbose image.png   # ImageMagick
```

Look for:
- Unusual or non-standard chunks
- `tEXt` / `zTXt` chunks with hidden messages
- Corrupted or duplicated chunks
- Modified IHDR (wrong dimensions hiding part of the image)

### 7. Dimension manipulation check
Sometimes the IHDR chunk reports smaller dimensions than the actual image data:
```bash
# Check reported dimensions
pngcheck image.png

# Try modifying IHDR height value in hex editor
# If image "grows" after fixing, data was hidden by cropping
```

## Common Hiding Techniques

| Technique | Detection method |
|-----------|-----------------|
| Data after IEND | `binwalk`, `hexdump` tail |
| LSB steganography | `zsteg`, `stegsolve` |
| Custom text chunks | `pngcheck -v`, `exiftool` |
| Modified dimensions (IHDR) | `pngcheck`, manual hex editing |
| Appended ZIP/RAR | `binwalk -e` |
| Color palette manipulation | `stegsolve` (analyze by plane) |

## PNG vs JPEG: Forensics Differences

| Aspect | PNG | JPEG |
|--------|-----|------|
| Compression | Lossless | Lossy |
| End marker | IEND chunk | `FF D9` |
| Steg tool | `zsteg` | `steghide` |
| LSB steg | Clean (no compression artifacts) | Unreliable (lossy compression) |
| Metadata | PNG chunks + EXIF | EXIF + JFIF |
| Transparency | Supported (alpha channel) | Not supported |

## References

- `tools/forensics-tools.md`
- `forensics/forensics-workflow.md`
- `forensics/image-forensics-jpeg.md`
