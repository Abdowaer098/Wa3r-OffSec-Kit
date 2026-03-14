# Forensics Investigation Workflow

## Overview
A structured workflow for approaching digital forensics challenges, whether in CTF competitions or real-world investigations. This document covers the standard analysis pipeline from evidence handling through artifact extraction and reporting.

## Phase 1: Initial Evidence Assessment

### Identify what you have
```bash
file evidence.*          # Determine file type
ls -la evidence.*        # Check file size
md5sum evidence.*        # Record hash for integrity
sha256sum evidence.*     # More secure hash
```

Before doing anything else, **hash the evidence** and record it. This proves you have not modified the original.

### Determine the evidence type

| Type | Examples | Next steps |
|------|----------|------------|
| Disk image | `.raw`, `.img`, `.dd`, `.E01` | Mount, partition analysis |
| Memory dump | `.mem`, `.dmp`, `.raw` | Volatility analysis |
| Image file | `.jpg`, `.png`, `.bmp` | Metadata, steganography |
| Network capture | `.pcap`, `.pcapng` | Wireshark analysis |
| Document | `.pdf`, `.docx`, `.xlsx` | Metadata, embedded objects |
| Archive | `.zip`, `.rar`, `.7z`, `.tar.gz` | Extract, analyze contents |
| Binary/unknown | No extension or misleading | `file`, `hexdump`, `strings` |

## Phase 2: Metadata Extraction

```bash
# General metadata
exiftool evidence_file

# For images specifically
exiftool -a -u -g1 image.jpg    # All tags, unknown tags, grouped

# For documents
exiftool document.pdf
```

**What to look for:**
- Author names, software versions, timestamps
- GPS coordinates (images)
- Hidden comments, descriptions
- Embedded thumbnails that differ from the main image
- Modification timestamps that seem suspicious

## Phase 3: String Analysis

```bash
# Basic string extraction
strings evidence_file > strings_output.txt

# Search for specific patterns
strings evidence_file | grep -i "flag\|password\|secret\|key\|http"

# Unicode strings
strings -e l evidence_file    # 16-bit little-endian
strings -e b evidence_file    # 16-bit big-endian
```

## Phase 4: Binary / Hex Analysis

```bash
# Formatted hex view
hexdump -C evidence_file | head -100

# Check magic bytes (first few bytes identify file type)
xxd evidence_file | head -5

# Look for end-of-file markers
hexdump -C evidence_file | tail -20
```

**Key magic bytes:**

| File type | Magic bytes (hex) |
|-----------|------------------|
| PNG | `89 50 4E 47 0D 0A 1A 0A` |
| JPEG | `FF D8 FF` |
| ZIP/DOCX/XLSX | `50 4B 03 04` |
| PDF | `25 50 44 46` |
| GIF | `47 49 46 38` |
| ELF | `7F 45 4C 46` |
| RAR | `52 61 72 21` |

## Phase 5: Embedded File Detection

```bash
# Scan for embedded files
binwalk evidence_file

# Extract embedded files
binwalk -e evidence_file

# Recursive extraction
binwalk -eM evidence_file
```

Check for:
- Files appended after the end-of-file marker (especially in PNG after `IEND`)
- Nested archives (ZIP inside an image)
- Encrypted or compressed embedded payloads

## Phase 6: Steganography Detection (for images)

### JPEG files
```bash
steghide info image.jpg                    # Check for embedded data
steghide extract -sf image.jpg             # Try empty passphrase
steghide extract -sf image.jpg -p "pass"   # Try known password
```

### PNG/BMP files
```bash
zsteg image.png           # Comprehensive steg detection
zsteg -a image.png        # Aggressive mode (all methods)
```

### General
```bash
# Check for data in specific color channels
stegsolve                  # GUI tool for image steg analysis
```

## Phase 7: Disk Image Analysis

```bash
# List partitions
fdisk -l disk.img
mmls disk.img

# Mount the image
mount -o loop,ro disk.img /mnt/evidence

# File carving from disk image
foremost -i disk.img -o recovered_files/
scalpel disk.img -o carved_files/

# Timeline analysis
fls -r -m "/" disk.img > timeline.body
mactime -b timeline.body > timeline.csv
```

## Phase 8: Reporting

Document your findings:
1. **Evidence identification**: Hash, size, type
2. **Tools used**: List each tool and version
3. **Findings**: What was discovered, with timestamps
4. **Chain of evidence**: Each step that led to the finding
5. **Conclusions**: What the evidence tells us

## Quick Decision Tree

```
Start
  ├── Is it an image? → Metadata → Strings → Steg tools
  ├── Is it a disk image? → Mount → Browse → Carve files
  ├── Is it a PCAP? → Wireshark → Follow streams → Extract files
  ├── Is it a document? → Metadata → Embedded objects → Macros
  ├── Is it a binary? → file → strings → hexdump → binwalk
  └── Unknown? → file → hexdump (check magic bytes) → proceed based on type
```

## References

- `tools/forensics-tools.md`
- `forensics/image-forensics-jpeg.md`
- `forensics/image-forensics-png.md`
