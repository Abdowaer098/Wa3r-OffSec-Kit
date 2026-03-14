# Forensics Tools

## Overview
A reference guide to the core tools used in digital forensics and CTF-style image/file analysis. Each tool is listed with its purpose, key commands, and practical tips.

## Core Tools

### file
- **Purpose**: Identify file type based on magic bytes, not file extension.
- **Example Commands**:
  ```bash
  file mystery_image.png
  file -i data.bin          # Show MIME type
  ```
- **Usage Tips**:
  - Always start with `file` — the extension may be misleading.
  - Check if the file type matches what you expect (e.g., a `.png` that is actually a ZIP).

### strings
- **Purpose**: Extract printable ASCII/Unicode strings from binary files.
- **Example Commands**:
  ```bash
  strings suspicious_file > extracted_strings.txt
  strings -n 10 image.jpg   # Only strings ≥ 10 characters
  strings -e l image.png     # Little-endian 16-bit (Unicode)
  ```
- **Usage Tips**:
  - Pipe through `grep` to find specific patterns: `strings file | grep -i flag`
  - Often reveals hidden messages, URLs, credentials, or markers embedded in files.

### exiftool
- **Purpose**: Read and write metadata (EXIF, IPTC, XMP) from images and documents.
- **Example Commands**:
  ```bash
  exiftool image.jpg
  exiftool -all= image.jpg   # Strip all metadata
  exiftool -Comment image.png # Read just the Comment field
  ```
- **Usage Tips**:
  - Check for GPS coordinates, author names, software versions, and hidden comments.
  - In CTFs, flags are often hidden in metadata fields like `Comment`, `UserComment`, or `Description`.

### steghide
- **Purpose**: Extract or embed hidden data in JPEG and BMP files using steganography.
- **Example Commands**:
  ```bash
  steghide info image.jpg               # Check if data is embedded
  steghide extract -sf image.jpg        # Extract with empty passphrase
  steghide extract -sf image.jpg -p "password"  # Extract with passphrase
  steghide embed -cf cover.jpg -ef secret.txt   # Embed data
  ```
- **Usage Tips**:
  - Only works with JPEG and BMP formats.
  - Try empty passphrase first, then common passwords.
  - If you suspect steganography but steghide fails, try other tools (zsteg, binwalk).

### binwalk
- **Purpose**: Scan files for embedded files, firmware images, and compressed archives.
- **Example Commands**:
  ```bash
  binwalk image.png               # Scan for signatures
  binwalk -e image.png            # Extract embedded files
  binwalk --dd='.*' firmware.bin  # Extract everything
  ```
- **Usage Tips**:
  - Detects ZIP, RAR, gzip, tar, ELF binaries, and many other formats hidden inside files.
  - The `-e` flag automatically carves out detected embedded content.
  - Critical for firmware analysis and multi-layer file challenges.

### hexdump / xxd
- **Purpose**: View raw hex data of files for manual analysis.
- **Example Commands**:
  ```bash
  hexdump -C file.bin | head -50     # Formatted hex + ASCII view
  xxd file.bin | head -50            # Alternative hex viewer
  xxd -r modified.hex output.bin     # Reverse: hex back to binary
  ```
- **Usage Tips**:
  - Use `-C` flag for combined hex + ASCII output (much easier to read).
  - Pipe through `head` or `tail` to view specific portions.
  - Look for: magic bytes, `PK` (ZIP), `IEND` (PNG end marker), padding patterns.

### zsteg
- **Purpose**: Detect steganography in PNG and BMP files using various bit-plane methods.
- **Example Commands**:
  ```bash
  zsteg image.png            # Run all detection methods
  zsteg -a image.png         # Try all known methods (aggressive)
  zsteg -E "b1,rgb,lsb" image.png  # Extract specific bit-plane
  ```
- **Usage Tips**:
  - Specifically designed for PNG and BMP (steghide cannot handle PNG).
  - Detects LSB steganography, OpenStego, and various encoding schemes.
  - Run with `-a` flag for thorough analysis when you suspect hidden data.

### foremost / scalpel
- **Purpose**: File carving — recover files from disk images or corrupted data.
- **Example Commands**:
  ```bash
  foremost -i disk.img -o output_dir/
  scalpel -c scalpel.conf disk.img -o recovered/
  ```
- **Usage Tips**:
  - Works on raw disk images, memory dumps, and corrupted files.
  - Carves files based on header/footer signatures.

## Quick Reference: Analysis Order

Start every forensics challenge or investigation with this sequence:

```bash
# 1. Identify the file
file mystery_file

# 2. Check metadata
exiftool mystery_file

# 3. Extract strings
strings mystery_file | head -100

# 4. Check for embedded files
binwalk mystery_file

# 5. View hex for anomalies
hexdump -C mystery_file | head -50

# 6. Try steganography tools (if image)
steghide info mystery_file    # For JPEG
zsteg mystery_file            # For PNG
```

## Text Processing Helpers

Useful for parsing forensics output:

| Command | Purpose |
|---------|---------|
| `cut -d ":" -f1` | Print only the first column (delimiter: `:`) |
| `cut -d ":" -f2` | Print only the second column |
| `cut -d ":" -f2,3` | Print columns 2 and 3 |
| `cut -d ":" -f2-` | Print from column 2 to end |
| `sort -u` | Sort and deduplicate |
| `grep -i "flag"` | Case-insensitive search for "flag" |

## PNG-Specific Notes

- PNG files end with an `IEND` chunk — data after `IEND` is not displayed but may contain hidden content.
- Use `binwalk` to detect files appended after `IEND`.
- `zsteg` is the go-to tool for PNG steganography.

## JPEG-Specific Notes

- JPEG metadata (EXIF) can contain GPS, camera info, and embedded thumbnails.
- `steghide` is the primary tool for JPEG steganography.
- Check for appended data after the JPEG end marker (`FF D9`).

## Related Techniques

- `forensics/forensics-workflow.md`
- `forensics/image-forensics-jpeg.md`
- `forensics/image-forensics-png.md`
