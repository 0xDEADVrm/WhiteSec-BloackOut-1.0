# Breaking the BLACKOUT: A Deep Dive into WHITESEC BLACKOUT 1.0 Reverse Engineering Challenges

### From corrupted ZIP archives and runtime-decrypted code to custom virtual machines, anti-debugging, SipHash, and controlled brute force

> **Spoiler Warning:** This article contains complete solutions, implementation details, recovered secrets, and flags for two reverse-engineering challenges from **WHITESEC BLACKOUT 1.0**. Read only if you are comfortable with spoilers.

---

## Introduction

**WHITESEC BLACKOUT 1.0** was designed as a hands-on Capture The Flag (CTF) experience focused on practical security problem solving.

Among the challenges I created for the event, two reverse-engineering problems were deliberately designed to move beyond the usual "`strings` → find flag" workflow:

- **The Blackout Protocol** — a layered artifact that combines file carving, multiple encoding/encryption layers, a custom virtual machine, and anti-debugging.
- **reverse_me_harder** — a protected ELF containing runtime-decrypted code, a flattened state machine, SipHash-based validation, anti-debugging logic, and a deliberately small brute-force search space.

Both challenges were built around the same idea:

> **The final secret should not exist in an obvious, static form. The solver has to understand the program's behavior.**

This write-up documents the intended solving path, the reasoning behind each stage, and the techniques that make these challenges work.

---

# Challenge 1 — The Blackout Protocol

## Challenge Profile

| Property | Details |
|---|---|
| Category | Reverse Engineering |
| Difficulty | Hard |
| Platform | Linux x86-64 |
| Primary Artifact | `blackout_protocol.zip` |
| Final Flag | `WHITESEC{th3_bl4ck0ut_w4s_n3v3r_0v3r}` |

The challenge begins with an artifact that appears to be a ZIP archive.

The important word here is **appears**.

The ZIP is intentionally damaged, and the archive structure cannot simply be extracted with `unzip`.

### Intended Attack Chain

```text
                 blackout_protocol.zip
                           │
                           ▼
                  Corrupted ZIP structure
                           │
                           ▼
                Carve local ZIP headers
                           │
                           ▼
              ┌─────────────────────────┐
              │ README.txt              │
              │ manifest.dat            │
              └─────────────────────────┘
                           │
                           ▼
                 Recover operation key
                       BLACKOUT
                           │
                           ▼
                  GZIP decompression
                           │
                           ▼
                    Base64 decode
                           │
                           ▼
                  Repeating-key XOR
                    key = BLACKOUT
                           │
                           ▼
                    recovery_node
                       ELF binary
                           │
                           ▼
                 Custom VM interpreter
                           │
                           ▼
                  Recover access key
               Bl4ck0ut_Acc3ss_2026
                           │
                           ▼
                    Run the binary
                           │
                           ▼
        WHITESEC{th3_bl4ck0ut_w4s_n3v3r_0v3r}
```

---

## 1. Inspecting the ZIP

The first step is ordinary file identification:

```bash
file blackout_protocol.zip
```

Trying to extract it:

```bash
unzip blackout_protocol.zip
```

fails because the archive's central directory/end-of-central-directory structure has been damaged.

This is intentional.

A common mistake is to stop at the archive parser and assume that a corrupt ZIP means the contents are unrecoverable.

ZIP files are structured containers. Even when the central directory is damaged, individual file entries may still contain useful local headers and compressed data.

---

## 2. Looking for ZIP Local File Headers

A ZIP local file header begins with the signature:

```text
PK\x03\x04
```

Instead of trusting the archive's directory, we can search for those signatures directly.

For example:

```bash
grep -oba $'PK\x03\x04' blackout_protocol.zip
```

The recovered offsets were:

```text
0
133
```

That immediately suggested that there were still two embedded file records.

The recovered files were:

```text
README.txt
manifest.dat
```

### Why this works

The ZIP central directory is mainly an index.

Individual entries contain their own local headers. Therefore, corruption of the directory does not necessarily destroy every embedded file.

This is a useful general forensic principle:

> **When a parser fails, inspect the underlying file format manually.**

---

## 3. Recovering the First Clue

The carved `README.txt` contains the next instruction.

The important clue is that the key is the **codename of the operation**, written in uppercase.

The operation name is:

```text
BLACKOUT
```

At this point, `BLACKOUT` becomes the working key for the next layer.

This is an important design choice: the first recovered file does not contain the flag. It contains the information required to interpret the second artifact.

---

# 4. Identifying `manifest.dat`

The filename is deliberately misleading.

Check the file:

```bash
file manifest.dat
xxd -l 16 manifest.dat
```

The beginning contains:

```text
1f 8b 08
```

Those bytes identify a GZIP stream.

So the correct interpretation is not:

```text
manifest.dat = arbitrary DAT file
```

but:

```text
manifest.dat = gzip-compressed data
```

Decompress it:

```bash
gunzip manifest.dat
```

This produces:

```text
stage2.txt
```

---

# 5. Base64 Layer

The resulting file contains Base64-encoded data.

Decode it:

```bash
base64 -d stage2.txt > stage3.bin
```

Now we have a binary blob, but it is still not the final executable.

This is where the previously recovered key becomes useful.

---

# 6. Repeating-Key XOR

The next layer uses repeating-key XOR with:

```text
BLACKOUT
```

Conceptually:

```python
data[i] ^= key[i % len(key)]
```

A minimal decoder looks like:

```python
from pathlib import Path

data = Path("stage3.bin").read_bytes()
key = b"BLACKOUT"

decoded = bytes(
    byte ^ key[i % len(key)]
    for i, byte in enumerate(data)
)

Path("recovery_node").write_bytes(decoded)
```

Now inspect the result:

```bash
file recovery_node
```

It identifies as an ELF executable.

We have finally reached the actual reverse-engineering target.

---

# 7. Initial ELF Reconnaissance

Start with the usual tools:

```bash
file recovery_node
strings recovery_node
readelf -h recovery_node
readelf -S recovery_node
```

At first glance, the binary does not simply reveal a password or flag.

Running it also does not immediately expose the secret.

This is intentional.

The challenge now transitions from **artifact recovery** to **program analysis**.

---

# 8. Anti-Debugging: `ptrace`

Static analysis of `main` reveals a call to:

```c
ptrace(PTRACE_TRACEME, ...)
```

This is a classic anti-debugging mechanism.

Under normal execution, the call succeeds.

When a debugger is already attached, the behavior changes, allowing the program to detect that it is being debugged.

The important lesson is not merely "patch `ptrace`."

The real lesson is:

> **When a binary behaves differently under a debugger, investigate anti-analysis logic before assuming your debugger is broken.**

For analysis, one possible approach is to neutralize the check.

For example, an analyst can patch the conditional branch around the `ptrace` result, or use a controlled `LD_PRELOAD` stub:

```c
long ptrace(int request, ...)
{
    return 0;
}
```

Compile:

```bash
gcc -shared -fPIC -o fake_ptrace.so fake_ptrace.c
```

Then:

```bash
LD_PRELOAD=./fake_ptrace.so gdb ./recovery_node
```

This is useful for controlled local analysis of the challenge binary.

---

# 9. Finding the Custom Virtual Machine

The most interesting part of the binary is a function referred to during analysis as:

```text
vm_check
```

Rather than comparing the input directly against a string, the program interprets a custom instruction stream.

The VM has the following relevant opcodes:

| Opcode | Operation |
|---:|---|
| `0x01` | `IN` |
| `0x02` | `XOR k` |
| `0x03` | `ADD k` |
| `0x04` | `ROL n` |
| `0x05` | `CMP v` |
| `0x06` | `HALT` |

The VM code is itself decrypted at runtime using an 8-byte XOR key stored as `vm_key`.

That gives us another layer:

```text
ELF
 ↓
VM interpreter
 ↓
runtime VM-code decryption
 ↓
instruction stream
 ↓
per-character validation
```

---

# 10. Reversing the Character Check

For each character, the VM effectively performs:

```text
IN
XOR k
ROL n
CMP v
```

If the input character is `x`, the transformed value is:

```text
ROL(x XOR k, n) = v
```

To recover `x`, simply invert the operations in reverse order:

```text
x XOR k = ROR(v, n)

x = ROR(v, n) XOR k
```

Therefore:

```python
input[i] = ror(v, n) ^ k
```

The challenge derives its per-character parameters as:

```python
k = (i * 37 + 11) & 0xff
n = (i % 7) + 1
```

This is a classic example of why reverse engineering is often about **inverting transformations**, not brute-forcing everything.

If a transformation is reversible, solve it algebraically.

---

# 11. Recovering the Access Key

Applying the inverse transformation to every VM comparison value produces:

```text
Bl4ck0ut_Acc3ss_2026
```

This is the access key expected by the executable.

At this point, the layered challenge has been reduced to a simple logical flow:

```text
Corrupted container
        ↓
Carving
        ↓
BLACKOUT
        ↓
GZIP
        ↓
Base64
        ↓
XOR
        ↓
ELF
        ↓
VM analysis
        ↓
Access key
```

Running the binary with the recovered key reveals the final flag.

---

# 12. Flag Reveal

The recovered flag is:

```text
WHITESEC{th3_bl4ck0ut_w4s_n3v3r_0v3r}
```

The important point is that the flag is not present as a convenient plaintext string in the initial artifact.

The solver must reconstruct the execution path.

---

## Why The Blackout Protocol Works as a Challenge

The challenge deliberately combines several independent skills:

- ZIP structure awareness
- File carving
- Magic-byte identification
- GZIP decompression
- Base64 decoding
- Repeating-key XOR
- ELF analysis
- Anti-debugging recognition
- Custom VM reversing
- Bitwise rotation
- Algebraic inversion

Each layer answers a question raised by the previous layer.

That makes the challenge feel like a chain rather than a collection of unrelated tricks.

---

# Challenge 2 — `reverse_me_harder`

## Challenge Profile

| Property | Details |
|---|---|
| Category | Reverse Engineering |
| Difficulty | Hard |
| Architecture | Linux x86-64 |
| Binary Type | PIE, dynamically linked, stripped ELF |
| Final Flag | `WHITESEC{b7f3a1}` |

The second challenge takes a different approach.

Instead of hiding the binary behind several external file formats, the protection is embedded **inside the executable itself**.

The solver has to discover code that is encrypted on disk, understand how it is unpacked at runtime, untangle a flattened state machine, recover a cryptographic check, account for anti-debugging behavior, and finally solve a deliberately constrained search problem.

### Intended Attack Chain

```text
                 reverse_me_harder
                         │
                         ▼
                 Stripped PIE ELF
                         │
                         ▼
                 Initial reconnaissance
                         │
                         ▼
               Encrypted code section
                         │
                         ▼
              Runtime mprotect + XOR
                         │
                         ▼
                 Decrypted code
                         │
                         ▼
              Flattened state machine
                         │
                         ▼
                Real validation logic
                         │
                         ▼
                    SipHash-2-4
                         │
                         ▼
                 Derived 16-byte key
                         │
                         ▼
                  Anti-debug taint
                         │
                         ▼
             6-character search space
                         │
                         ▼
                     b7f3a1
                         │
                         ▼
              WHITESEC{b7f3a1}
```

---

# 1. Start With Basic Reconnaissance

First identify the binary:

```bash
file reverse_me_harder
```

The important properties are:

```text
Linux x86-64
PIE
dynamically linked
stripped
```

A stripped binary removes many convenient symbol names, so we cannot rely on function names to tell us what the program is doing.

Next:

```bash
strings reverse_me_harder
```

There is a tempting clue:

```text
WHITESEC{
```

But there is no complete flag.

This is deliberate.

A useful reverse-engineering rule is:

> **A string that looks like a flag format is evidence, not proof that the flag is stored nearby.**

---

# 2. Inspecting `main`

Disassembling the entry logic reveals several interesting operations:

```text
ptrace
fgets
unpack/decrypt routine
real_check
```

The `ptrace` call immediately suggests anti-debugging.

The unpack/decrypt routine is even more interesting.

The binary contains a section referred to during analysis as:

```text
enc_text
```

This section contains encrypted code.

So the code we need is not directly executable in its useful form on disk.

---

# 3. Runtime Code Decryption

The program changes the memory permissions of the encrypted region using:

```c
mprotect(..., PROT_READ | PROT_WRITE | PROT_EXEC)
```

Then it decrypts the region using a repeating 16-byte XOR key.

Conceptually:

```python
for i in range(length):
    memory[i] ^= key[i % 16]
```

After decryption, execution jumps into the now-readable code.

When the check finishes, the program re-encrypts the region.

This gives the binary a self-protecting behavior:

```text
Encrypted code on disk
        ↓
mprotect()
        ↓
Runtime XOR decrypt
        ↓
Execute
        ↓
Re-encrypt
```

### Why this works

Static tools see the encrypted representation.

The actual validation logic only exists in plaintext during execution.

This is a simple but effective form of runtime code protection and forces the analyst to examine the unpacking routine or capture the decrypted memory.

---

# 4. The Flattened State Machine

Once the code is decrypted, the next obstacle is control flow.

Instead of a clean sequence of:

```c
if (...)
    ...
else
    ...
```

the validator uses a flattened state machine.

Conceptually:

```text
             ┌──────────────┐
             │ Dispatcher   │
             └──────┬───────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
      State A     State B     State C
        │           │           │
        └───────┬───┴───────┬───┘
                ▼           ▼
              Dead / opaque states
                    │
                    ▼
                Real check
```

Many states are irrelevant.

Some transitions are governed by opaque predicates or deliberately dead branches.

The goal is to identify the **semantic path**, not manually understand every branch.

This is a common reverse-engineering lesson:

> **Control-flow complexity does not necessarily imply algorithmic complexity.**

---

# 5. Recovering the Real Validation Logic

After untangling the state machine, the actual constraints become clear.

The candidate input:

- must be exactly 16 bytes,
- begins with:

```text
WHITESEC{
```

- ends with:

```text
}
```

Therefore the unknown portion is only:

```text
6 characters
```

The allowed character set is:

```text
abcdefghijklmnopqrstuvwxyz0123456789
```

So the search space is:

```text
36^6 = 2,176,782,336
```

That is large enough to make naive scripting unattractive, but small enough to become practical with a compiled, optimized checker.

---

# 6. SipHash-2-4

The core validation uses **SipHash-2-4**.

The program derives a 16-byte key from several internal constants.

The recovered constants are:

```text
key_part_a =
9f 2c 71 e4 03 88 5a 1d

key_part_b =
66 b0 2e 47 d9 12 c5 3a

key_rot =
03 01 04 01 05 09 02 06
```

The resulting derived key is:

```text
fc 58 17 c9 60 11 69 47
65 b1 2a 46 dc 1b c7 3c
```

The target SipHash digest, interpreted in little-endian form, is:

```text
fe bd 93 e9 1f e6 c6 27
```

The validation therefore looks conceptually like:

```text
candidate
   │
   ▼
16-byte format validation
   │
   ▼
derive 16-byte key
   │
   ▼
SipHash-2-4(candidate, key)
   │
   ▼
compare against target digest
   │
   ├── match → accept
   └── mismatch → reject
```

---

# 7. Understanding the Key Derivation

The important reversing task is not simply finding constants.

We need to understand how they interact.

The program combines:

```text
key_part_a
key_part_b
key_rot
g_taint
```

to produce the final 16-byte SipHash key.

This is why copying constants from the binary without understanding their role is insufficient.

A debugger or modified execution environment can change the derived key through `g_taint`.

---

# 8. Anti-Debugging and `g_taint`

The binary again uses:

```c
ptrace(PTRACE_TRACEME, ...)
```

But here the anti-debugging behavior is tied directly to the cryptographic validation.

The logic is effectively:

```text
normal execution
      │
      ▼
g_taint = 0x00
      │
      ▼
normal derived key
```

Whereas a detected debugger causes:

```text
debugger detected
      │
      ▼
g_taint = 0xFF
      │
      ▼
different derived key
      │
      ▼
valid candidate rejected
```

This is a stronger anti-debugging design than simply terminating the process.

The program can continue running while quietly producing the wrong answer.

That makes the analyst question the correctness of an otherwise apparently valid reverse-engineering result.

---

# 9. Why Brute Force Becomes Reasonable

Once the validator is fully understood, the remaining unknown is only six characters:

```text
[a-z0-9]{6}
```

The search space is:

```text
36^6 = 2,176,782,336
```

A pure Python implementation would be unnecessarily slow for this workload.

A better approach is to compile the checker and optimize the search.

Conceptually:

```python
for candidate in candidates:
    flag = "WHITESEC{" + candidate + "}"

    if siphash24(flag, derived_key) == target:
        print(flag)
        break
```

In practice, the challenge is best approached with a compiled implementation of SipHash-2-4 and a tight candidate loop.

The key insight is:

> **Brute force should come after reducing the problem.**

Trying to brute-force an unknown 16-byte value would be absurd.

Brute-forcing six characters after reversing the format, key derivation, and digest is a controlled search problem.

---

# 10. Recovered Candidate

The search yields:

```text
b7f3a1
```

Therefore the complete candidate is:

```text
WHITESEC{b7f3a1}
```

---

# 11. Flag Reveal

The final flag is:

```text
WHITESEC{b7f3a1}
```

---

# Why `reverse_me_harder` Works as a Challenge

This challenge combines several reverse-engineering concepts:

- stripped ELF analysis,
- PIE-aware reversing,
- runtime code unpacking,
- executable memory permissions,
- XOR-based code decryption,
- control-flow flattening,
- opaque/dead states,
- cryptographic validation,
- key derivation,
- anti-debugging taint,
- and constrained brute force.

The intended experience is not "find one clever trick."

It is:

```text
Observe
  ↓
Hypothesize
  ↓
Trace
  ↓
Simplify
  ↓
Recover the algorithm
  ↓
Solve the reduced problem
```

---

# Comparing the Two Challenges

| Aspect | The Blackout Protocol | reverse_me_harder |
|---|---|---|
| Primary theme | Layered artifact recovery | Protected executable |
| File challenge | Corrupted ZIP | Encrypted code section |
| Encoding layers | GZIP → Base64 → XOR | Runtime XOR |
| Main reversing target | Custom VM | Flattened state machine |
| Anti-debugging | `ptrace` | `ptrace` + taint |
| Cryptography | Reversible bitwise transformations | SipHash-2-4 |
| Final technique | Algebraic inversion | Controlled brute force |
| Flag | `WHITESEC{th3_bl4ck0ut_w4s_n3v3r_0v3r}` | `WHITESEC{b7f3a1}` |

---

# Lessons Learned

## 1. Don't Trust File Extensions

A file called:

```text
manifest.dat
```

can still be a GZIP stream.

Always inspect magic bytes and actual content.

```bash
file manifest.dat
xxd -l 16 manifest.dat
```

---

## 2. Parser Failure Is Not the End

A broken ZIP parser does not necessarily mean the data is gone.

When metadata is damaged, look for structural signatures:

```text
PK\x03\x04
```

This approach is broadly useful in digital forensics.

---

## 3. Reverse Transformations Instead of Guessing

The VM's validation:

```text
x → XOR → ROL → compare
```

is reversible.

Instead of guessing `x`, invert the operations:

```text
compare
  ↓ ROR
  ↓ XOR
input
```

Understanding the mathematical structure often eliminates brute force completely.

---

## 4. Runtime-Decrypted Code Changes the Static Analysis Game

If a binary decrypts its code only after startup, the analyst needs to investigate:

- the decrypt routine,
- memory permissions,
- runtime memory,
- and the point where execution enters the decrypted region.

Static strings alone are insufficient.

---

## 5. Anti-Debugging Can Affect Logic, Not Just Execution

Both challenges demonstrate a subtle point.

Anti-debugging does not always mean:

```text
if debugger:
    exit()
```

It can instead mean:

```text
if debugger:
    alter_state()
```

That second pattern can be much harder to notice because the program continues to run.

---

## 6. Reduce the Search Space Before Brute Force

For `reverse_me_harder`, the raw search problem initially looks intimidating.

After reversing the validation logic, it becomes:

```text
WHITESEC{ + 6 lowercase alphanumeric characters + }
```

That reduction is what makes a controlled brute-force approach reasonable.

---

# Challenge Author Perspective

As the author of these challenges, I wanted both problems to reward **curiosity and methodical analysis** rather than a single tool or magic command.

For **The Blackout Protocol**, the goal was to create a chain where every recovered layer provides the clue needed for the next one:

```text
Damaged ZIP
    ↓
Carving
    ↓
README
    ↓
BLACKOUT
    ↓
GZIP
    ↓
Base64
    ↓
XOR
    ↓
ELF
    ↓
VM
    ↓
Access key
    ↓
Flag
```

The challenge intentionally asks the solver to move between different areas of security analysis: file formats, encoding, cryptography, binary analysis, and program execution.

For **reverse_me_harder**, the design philosophy was different.

I wanted the executable itself to be the puzzle.

The solver first encounters a stripped binary, then discovers that the interesting code is encrypted, then discovers that the decrypted code is intentionally difficult to follow, and finally reaches a cryptographic check.

The final brute-force step is deliberately not the main challenge.

The main challenge is getting to a point where brute force becomes the **correct engineering decision**.

That distinction is important.

A good CTF challenge should ideally teach something even after the flag has been recovered.

---

# A Practical Reverse-Engineering Workflow

These two challenges also demonstrate a reusable workflow for future CTF problems.

### Step 1 — Identify the artifact

```bash
file <target>
```

### Step 2 — Inspect obvious metadata

```bash
strings <target>
readelf -h <target>
readelf -S <target>
```

### Step 3 — Look for format signatures

Do not trust filenames.

Check magic bytes:

```bash
xxd <target>
```

### Step 4 — Understand the execution model

Ask:

- Is code unpacked?
- Is memory modified?
- Is there anti-debugging?
- Is input transformed?
- Is there a custom interpreter?

### Step 5 — Recover the actual algorithm

Don't stop at:

```text
"This looks obfuscated."
```

Reduce it to:

```text
input → transformation → comparison
```

### Step 6 — Choose the cheapest solving method

If the transformation is reversible:

```text
invert it
```

If the remaining search space is small:

```text
brute-force it
```

The right solution is usually the one that minimizes unnecessary work.

---

# Final Takeaways

The most important lesson from these challenges is that reverse engineering is rarely about a single tool.

It is about building a model of what the program is doing.

For **The Blackout Protocol**, that model was:

```text
corrupted archive
→ recovered files
→ layered decoding
→ executable
→ VM
→ reversible checks
→ access key
→ flag
```

For **reverse_me_harder**, it was:

```text
stripped ELF
→ runtime decryption
→ recovered code
→ flattened control flow
→ cryptographic validation
→ anti-debug state
→ reduced search space
→ flag
```

In both cases, the flag became straightforward only after the surrounding complexity had been removed.

That is one of the most satisfying parts of reverse engineering:

> **The program looks complicated until you understand it. Then the complexity starts to collapse.**

---

# Conclusion

**WHITESEC BLACKOUT 1.0** was built around the idea of making participants investigate rather than guess.

These two reverse-engineering challenges were designed to reinforce that mindset from different directions.

**The Blackout Protocol** focuses on layered recovery and custom execution logic.

**reverse_me_harder** focuses on runtime protection, control-flow complexity, cryptographic validation, and disciplined brute force.

Together, they demonstrate a broader security principle:

> **Don't fight the complexity blindly. Decompose it.**

Identify the layer.

Understand what it protects.

Reverse the transformation.

Reduce the problem.

Then solve what remains.

That is the mindset that turns a seemingly impossible binary into a solvable CTF challenge.

---

## Flags

For completeness, the final recovered flags were:

```text
The Blackout Protocol
WHITESEC{th3_bl4ck0ut_w4s_n3v3r_0v3r}

reverse_me_harder
WHITESEC{b7f3a1}
```

---

### About WHITESEC BLACKOUT 1.0

**WHITESEC BLACKOUT 1.0** was a practical CTF focused on hands-on cybersecurity skills, including web exploitation, cryptography, forensics, OSINT, Linux, reverse engineering, steganography, and binary analysis.

The reverse-engineering challenges discussed in this article were created to encourage participants to move beyond automated flag hunting and understand how software actually behaves.

**Hack • Learn • Compete.**

---

*Written as a post-event technical write-up for WHITESEC BLACKOUT 1.0.*
