# CTF Write-Up: LegoClicker — `liblegocore.so`

**Category:** Android Native Reverse Engineering  
**Flag:** `UMASS{br1ck_by_br1ck_y0u_r3ach3d_th3_t0p}`

---

## Overview

This challenge ships an Android APK whose logic lives entirely inside a native ARM64 shared library, `liblegocore.so`. Three JNI functions are registered dynamically at runtime. The flag is produced by a custom bytecode virtual machine that applies a chain of transformations to interleaved ciphertext arrays. The challenge layers multiple red herrings — a troll flag, a fake timing-based key system, and ESIL-hostile jump tables — to discourage straightforward emulation. The actual key material is static and lives in `.rodata`.

---

## Step 1 — Initial Reconnaissance

Unpack the APK and identify the relevant binary:

```
lib/arm64-v8a/liblegocore.so   (338 KB, stripped ELF64 ARM)
lib/x86_64/liblegocore.so      (also present, but addresses differ)
```

Load the ARM64 variant in Radare2 and run full analysis:

```
r2 liblegocore_arm64.so
[0x00000000]> aaa
```

Locate `JNI_OnLoad` and follow the dynamic JNI registration to find the three exported Java-callable functions:

|Virtual Address|Java Method|
|---|---|
|`0x22e30`|`verifyScore()`|
|`0x23000`|`getFlag()`|
|`0x23150`|(score helper)|

---

## Step 2 — The Troll Flag

`fcn.00023880`, reachable from `getFlag()`, immediately looks interesting: it builds a string from three byte arrays using a simple modulo-based lookup.

Decoding it yields:

```
BHREV{fAk3_flAG_wr0ng_s3ss10n}
```

This is a classic CTF troll flag — deliberately placed to waste time. Notice it uses `BHREV` (ROT-7 of `UMASS`). The real flag mechanism is elsewhere.

---

## Step 3 — Discovering the Custom VM

`fcn.00023c90`, called from the deeper path of `fcn.00022d3c`, is the real flag engine. It is a custom bytecode Virtual Machine (VM) that processes **41 bytes** of ciphertext to produce the flag.

### VM Architecture

The VM runs a fixed opcode pipeline for each of the 41 output bytes:

```
for i in 0..40:
    byte ← keystream_select(i)          // pick from 3 interleaved arrays
    byte ← DEAD_handler(byte, i)
    byte ← C0DE_handler(byte, i)
    byte ← CAFE_handler(byte, i)
    byte ← F00D_handler(byte)
    byte ← ACED_handler(byte, i)
    byte ← D07E_handler(byte, i)
    output[i] ← byte                    // flag character
```

The opcodes are stored as 16-bit constants in registers (`w25=0xDEAD`, `w25=0xBEEF`, etc.) and dispatched via comparison chains — not a table — making ESIL emulation crash on the empty jump table at `0x59000`.

### Keystream Interleaving (fcn.00021ff0)

The input bytes are read from three separate arrays using a round-robin pattern:

|Index `i`|Source Array|
|---|---|
|i % 3 == 0|`arr0` @ `0x14108`|
|i % 3 == 1|`arr1` @ `0x141ba`|
|i % 3 == 2|`arr2` @ `0x141c8`|

Within each array the position is `floor(i / 3)`. The selection is implemented via the multiply-shift trick for division by 3 (`× 0xAAAAAAAB >> 33`).

---

## Step 4 — Analyzing the Worker Functions

`fcn.00022064` initializes the VM's jump table at runtime, revealing the real addresses of the five worker functions:

|Opcode|Worker Address|Operation|
|---|---|---|
|`DEAD`|`0x221b4`|`byte XOR key_dead[(i+1) & 7]`|
|`C0DE`|`0x221ec`|`byte - key_dead[i & 7]`|
|`CAFE`|`0x22214`|`byte XOR key_cafe[i & 7]`|
|`F00D`|`0x22248`|`rotl8(byte, 5)` (pure, no key)|
|`ACED`|`0x2226c`|`byte XOR key_aced[i & 7]`|

The `D07E` handler (inline in the main VM loop) applies `byte XOR key_d07e[i % 4]`.

The XOR operations use the identity `(A | B) − (A & B) = A XOR B` as an obfuscation layer but are semantically equivalent to XOR.

`F00D` is the only operation with no key — it performs a constant circular left-rotation of 5 bits (equivalent to `ror8(byte, 3)`).

---

## Step 5 — Breaking the "Timing Key" Red Herring

The most important misdirection: JNI_OnLoad calls `clock_gettime(CLOCK_REALTIME)` and `clock_gettime(CLOCK_MONOTONIC)`, mixes the results through several XOR/ORN/shift operations, and stores 4 bytes to `0x5a020` (the `D07E` key table). This makes it appear the keys are non-deterministic.

**The reality:** `fcn.00022cdc`, called _before_ the timing code, loads the three main key tables by copying 8-byte `double` values directly from `.rodata` into BSS:

```
ldr d0, [x8, 0xc0]  → 0x140c0 → store → 0x59f90  (key_aced)
ldr d1, [x9, 0xf0]  → 0x140f0 → store → 0x59f98  (key_cafe)
ldr d2, [x10,0xe0]  → 0x140e0 → store → 0x59fa0  (key_dead/code)
```

The keys are **completely static**:

|Table|Source|Bytes|
|---|---|---|
|`key_dead`|`0x140e0`|`53 0a 7f 41 de 28 64 9c`|
|`key_cafe`|`0x140f0`|`20 67 c3 11 95 4e b2 39`|
|`key_aced`|`0x140c0`|`6c 1a 3f 88 2b 5d 71 04`|

As for `key_d07e`: the timing computation collapses to `orn(x, x) + 1 XOR x = 0x0` for both clock readings at startup, yielding all-zero bytes. This is confirmed by known-plaintext: the pre-D07E stream already decodes to `UMASS{...}`, so `key_d07e = [0, 0, 0, 0]`.

---

## Step 6 — Final Decryption

With all keys resolved, the full Python solver is:

```python
with open("liblegocore_arm64.so", "rb") as f:
    data = f.read()

key_aced = list(data[0x140c0:0x140c8])  # [0x6c,0x1a,0x3f,0x88,0x2b,0x5d,0x71,0x04]
key_cafe = list(data[0x140f0:0x140f8])  # [0x20,0x67,0xc3,0x11,0x95,0x4e,0xb2,0x39]
key_dead = list(data[0x140e0:0x140e8])  # [0x53,0x0a,0x7f,0x41,0xde,0x28,0x64,0x9c]
key_d07e = [0, 0, 0, 0]

arr0 = list(data[0x14108:0x14108+14])
arr1 = list(data[0x141ba:0x141ba+14])
arr2 = list(data[0x141c8:0x141c8+13])

def get_byte(i):
    q = i // 3
    return [arr0, arr1, arr2][i % 3][q]

def xor_op(byte, key):
    return ((key | byte) - (key & byte)) & 0xFF

flag = []
for i in range(41):
    b = get_byte(i)
    b = xor_op(b, key_dead[(i+1) & 7])      # DEAD
    b = (b - key_dead[i & 7]) & 0xFF         # C0DE
    b = xor_op(b, key_cafe[i & 7])           # CAFE
    b = ((b << 5) | (b >> 3)) & 0xFF         # F00D  (rotl 5)
    b = xor_op(b, key_aced[i & 7])           # ACED
    b ^= key_d07e[i % 4]                     # D07E
    flag.append(b)

print(''.join(chr(c) for c in flag))
```

Output:

```
UMASS{br1ck_by_br1ck_y0u_r3ach3d_th3_t0p}
```

---

## Lessons Learned

|Technique|Purpose in Challenge|
|---|---|
|Dynamic JNI registration|Hide function names from string search|
|Troll flag (`BHREV{...}`)|Waste time, bait for early submitters|
|VM with opcode dispatch|Prevent trivial ESIL/angr emulation|
|Empty jump table at `0x59000`|Crash emulators that try to follow calls|
|`clock_gettime` key derivation|Appear non-deterministic statically|
|`fcn.00022d10` always-zero hash|Noise in the key derivation path|
|BSS key tables|Seem runtime-only; actually seeded from `.rodata`|

The core insight: **follow the data, not the code**. Rather than trying to emulate the VM end-to-end, tracing _where_ the key tables are written (`fcn.00022cdc`) collapses the entire timing red herring and makes the keys visible in plain `.rodata`.

