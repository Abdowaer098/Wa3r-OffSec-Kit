# Image Forensics: JPEG

## Overview
JPEG (Joint Photographic Experts Group) is the most common image format and a frequent target in forensics investigations and CTF challenges. This guide covers JPEG structure, metadata analysis, steganography detection, and common hiding techniques.

## JPEG File Structure

| Component | Description |
|-----------|-------------|
| `FF D8` | Start of Image (SOI) marker |
| `FF E0` / `FF E1` | APP0 (JFIF) / APP1 (EXIF) markers |
| Various segments | Quantization tables, Huffman tables, image data |
| `FF D9` | End of Image (EOI) marker |

**Key insight**: Data appended **after** `FF D9` is ignored by image viewers but may contain hidden files or messages.

## Analysis Workflow

### 1. Identify and validate
```bash
file image.jpg
# Should report: JPEG image data, ...

hexdump -C image.jpg | head -5
# Should start with: ff d8 ff
```

### 2. Extract metadata
```bash
exiftool image.jpg
exiftool -a -u -g1 image.jpg   # All tags, grouped
```

**Look for:**
- Camera make/model, GPS coordinates
- Software used to create/edit
- `Comment` and `UserComment` fields (common flag locations)
- Embedded thumbnail that differs from the main image
- Suspicious timestamps (creation vs modification)

### 3. Search for strings
```bash
strings image.jpg | grep -i "flag\|password\|secret\|key\|ctf"
strings -n 20 image.jpg   # Only long strings
```

### 4. Check for appended data
```bash
# View end of file
hexdump -C image.jpg | tail -20

# If data exists after FF D9, extract it
binwalk image.jpg
binwalk -e image.jpg   # Auto-extract embedded files
```

### 5. Steganography detection
```bash
steghide info image.jpg               # Check if data is embedded
steghide extract -sf image.jpg        # Try empty passphrase
steghide extract -sf image.jpg -p ""  # Explicit empty passphrase
```

If passphrase is unknown, try common passwords or use `stegcracker`:
```bash
stegcracker image.jpg /usr/share/wordlists/rockyou.txt
```

### 6. Visual analysis
- Compare with original (if available) for pixel-level differences.
- Check for unusual color patterns in specific regions.
- Use image editing tools to adjust levels/curves and reveal hidden messages.

## Common Hiding Techniques

| Technique | Detection method |
|-----------|-----------------|
| Data after EOI (`FF D9`) | `binwalk`, `hexdump` tail |
| EXIF metadata | `exiftool` |
| Steghide embedding | `steghide info` |
| LSB modification | Visual analysis, statistical tools |
| Embedded files in EXIF | `binwalk`, `exiftool -b -ThumbnailImage` |
| Comment field | `exiftool -Comment`, `strings` |

## References

- `tools/forensics-tools.md`
- `forensics/forensics-workflow.md`
- `forensics/image-forensics-png.md`
