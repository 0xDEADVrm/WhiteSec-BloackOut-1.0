# Breaking the BLACKOUT

## A Technical Deep Dive into WHITESEC BLACKOUT 1.0

> **From corrupted ZIP archives and runtime-decrypted code to custom virtual machines, anti-debugging, SipHash, and controlled brute force.**

**WHITESEC Cybersecurity Club · Panimalar Engineering College**  
**Post-Event Technical Write-Up · September 2026**

---

### Abstract

**WHITESEC BLACKOUT 1.0** was designed as a practical cybersecurity challenge environment in which participants were expected to investigate systems, identify weaknesses, reconstruct hidden data, and reason about program behavior.

This article presents a consolidated technical walkthrough of the challenge set, moving from foundational Web and Linux investigations to more advanced reverse-engineering problems. Particular attention is given to the design and solution methodology behind **The Blackout Protocol** and **`reverse_me_harder`**, two challenges built to reward systematic binary analysis rather than superficial string hunting.

The objective is not merely to document the flags. It is to explain the reasoning process behind the solves, why each technique works, and what the challenges were intended to teach.

---

> **Spoiler Warning:** This article contains complete solutions, implementation details, recovered secrets, and flags for two reverse-engineering challenges from **WHITESEC BLACKOUT 1.0**. Read only if you are comfortable with spoilers.

## Contents

1. Introduction
2. Foundational Web & Linux Challenges
3. The Blackout Protocol
4. `reverse_me_harder`
5. Comparative Analysis
6. Technical Lessons
7. Challenge-Author Perspective
8. Reusable Reverse-Engineering Methodology
9. Final Takeaways
10. Conclusion

---

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


---

### The Design Objective

The challenge set was structured around a progression of investigative depth:

```text
Foundational concepts
        ↓
Web & Linux investigation
        ↓
File / data analysis
        ↓
Steganography & OSINT
        ↓
Binary analysis
        ↓
Runtime protection
        ↓
Custom validation logic
        ↓
Reduced, verifiable solution
```

Rather than requiring a single specialized tool, the challenges were intended to teach participants how to select the right technique from the evidence available.

# Part I — Foundational Web & Linux Challenges

The WHITESEC BLACKOUT 1.0 challenge set also included foundational Web and Linux/File Investigation challenges designed to give participants practical exposure to session handling, directory enumeration, SQL injection, source-code analysis, process inspection, and nested archives.

This section incorporates the coordinator's challenge-design and solution walkthrough into the post-event technical documentation.

---

## Web Exploitation

## The Forgotten Session

**Category:** Web  
**Points:** 100  
**Concept:** Cookies and Session Logic

The application trusts information stored in the user's session/cookie to determine whether the user is ordinary or privileged.

### Intended Attack Chain

```text
Login / Application
       ↓
Inspect browser storage
       ↓
Locate session cookie
       ↓
Understand encoded session state
       ↓
Modify privileged state
       ↓
Refresh application
       ↓
Privileged access
       ↓
Flag
```

### Solution Walkthrough

Open the challenge URL and inspect the login page or application interface.

Use browser Developer Tools:

```text
Application / Storage → Cookies
```

or inspect the traffic with Burp Suite.

Look for a cookie similar to:

```text
session=...
```

The goal is to understand how the session value represents the user's role or application state.

Once the encoding/structure is understood, modify the session value so that the application interprets the user as privileged.

Refresh the application and observe that the authorization state has changed.

### Flag

```text
whitesec{session_detective}
```

### Learning Outcome

This challenge introduces:

- Cookies and sessions
- Client-side session state
- Session manipulation
- The danger of trusting user-controlled authorization data

A core security lesson is that sensitive authorization decisions should not blindly trust values that an attacker can manipulate.

---

## The Hidden Directory

**Category:** Web  
**Points:** 100  
**Concept:** Directory Enumeration

This challenge hides a resource that is not linked from the main webpage.

### Intended Attack Chain

```text
Target Website
      ↓
Inspect visible content/source
      ↓
Directory enumeration
      ↓
Discover hidden path
      ↓
Visit resource
      ↓
Flag
```

### Gobuster

A typical enumeration command is:

```bash
gobuster dir \
  -u http://TARGET \
  -w /usr/share/wordlists/dirb/common.txt
```

Alternatively, use FFUF:

```bash
ffuf \
  -u http://TARGET/FUZZ \
  -w <WORDLIST>
```

The enumeration reveals the hidden directory, for example:

```text
/hidden
```

Navigate to the discovered resource and retrieve the flag.

### Flag

```text
whitesec{Directory_enumeration_success}
```

### Learning Outcome

Participants learn:

- Directory enumeration
- Gobuster and FFUF
- Discovery of unlinked web resources
- Why "not linked" does not mean "secure"

---

## The Admin's Mistake

**Category:** Web  
**Points:** 100  
**Concept:** SQL Injection

The login form constructs a database query using unsanitized input.

### Intended Attack Chain

```text
Login form
    ↓
Test authentication
    ↓
Identify SQL injection
    ↓
Inject authentication bypass
    ↓
Database query altered
    ↓
Admin access
    ↓
Flag
```

A classic test payload is:

```text
' OR '1'='1'--
```

A vulnerable backend may construct a query logically similar to:

```sql
SELECT * FROM users
WHERE username = '' OR '1'='1'--'
AND password = '...';
```

Because:

```text
'1'='1'
```

is always true, the authentication condition can be bypassed when the backend is vulnerable to this form of injection.

### Flag

```text
whitesec{sql_1nj3ct10n_byp4ss_2026!}
```

### Learning Outcome

This challenge demonstrates:

- SQL injection
- Unsafe dynamic SQL construction
- Authentication bypass
- Why parameterized queries and prepared statements matter

The secure implementation should use parameterized database queries rather than concatenating user input into SQL.

---

## The Secret in the Source

**Category:** Web  
**Points:** 100  
**Concept:** Source Code Analysis

The flag is hidden in client-side source rather than the rendered page.

### Intended Attack Chain

```text
Webpage
   ↓
View source / Developer Tools
   ↓
Search HTML + JavaScript
   ↓
Inspect comments / hidden elements / variables
   ↓
Find embedded flag
```

Open the page source using:

```text
Ctrl + U
```

or:

```text
Right-click → View Page Source
```

Developer Tools can also be used.

Search for:

```text
flag
```

or:

```text
WHITESEC
```

Inspect:

- HTML comments
- JavaScript
- Hidden elements
- Variables
- Linked source files

The embedded flag can then be recovered.

### Flag

```text
WHITESEC{s0urc3_c0d3_1sn7_pr1v4t3}
```

### Learning Outcome

Participants learn:

- HTML source inspection
- Developer Tools
- Client-side code exposure
- Why secrets should never be embedded in frontend code

---

## Linux & File Investigation

## The Process That Knows Too Much

**Category:** Linux  
**Points:** 100  
**Concept:** Process Analysis

The flag is not stored in an obvious static file. Instead, relevant information is exposed through a running process.

### Intended Attack Chain

```text
Running System
      ↓
List processes
      ↓
Identify interesting PID
      ↓
Inspect /proc
      ↓
Follow process information
      ↓
Recover flag
```

List running processes:

```bash
ps aux
```

or:

```bash
ps -ef
```

Filter for an interesting process:

```bash
ps aux | grep <keyword>
```

Once a candidate PID is identified, inspect its `/proc` entry:

```bash
cat /proc/<PID>/cmdline
```

and:

```bash
ls -la /proc/<PID>/
```

Follow the clue surfaced by process inspection to retrieve the flag.

### Flag

```text
whitesec{pr0c3ss_1s_n3v3r_pr1v4te}
```

### Learning Outcome

This challenge introduces:

- Process enumeration
- PIDs
- `ps`
- The Linux `/proc` filesystem
- Information leakage through running processes

---

## Matryoshka

**Category:** Linux / File Investigation  
**Concept:** Nested Archives, Extraction, and Investigation

Named after Russian nesting dolls, this challenge contains an archive inside another archive, which contains another archive, and so on.

### Intended Attack Chain

```text
Outer Archive
     ↓
Extract
     ↓
Archive
     ↓
Extract
     ↓
Archive
     ↓
Extract
     ↓
...
     ↓
Final Layer
     ↓
Flag
```

Download the challenge archive, for example:

```text
matryoshka-challengefinal.zip
```

Extract the first layer:

```bash
unzip challenge.zip
```

Inspect the result:

```bash
ls -la
```

Identify the next archive and repeat:

```text
archive → extract → archive → extract → ...
```

Continue until the final layer is reached.

### Flag

```text
WHITESEC{n3st3d_arch1v3s_0n3_st3p_at_a_t1me}
```

### Learning Outcome

Participants practice:

- Archive extraction
- File-type identification
- Recursive investigation
- Linux command-line file handling
- Maintaining context while moving through multiple layers

---

# Tools Used Across the CTF

The challenge set intentionally uses tools that map closely to the concepts being taught.

## Web Security

```text
Browser Developer Tools
Burp Suite
Gobuster
FFUF
```

## Linux / File Investigation

```text
ls
find
grep
ps
cat
chmod
stat
unzip
```

## Reverse Engineering

```text
file
strings
readelf
xxd
GDB
GCC
Python
```

The exact tool is less important than understanding what information it provides.

For example:

```text
file     → identify the artifact
strings  → inspect printable data
readelf  → inspect ELF structure
xxd      → inspect raw bytes
gdb      → observe execution
grep     → locate structural signatures
```

---

# Design Philosophy Across the Challenge Set

The challenge set was deliberately built to expose participants to different layers of practical security analysis.

```text
                    WHITESEC BLACKOUT 1.0
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
       WEB                 LINUX              REVERSE
        │                    │                    │
   ┌────┼────┐          ┌────┴────┐       ┌─────┴─────┐
   ▼    ▼    ▼          ▼         ▼       ▼           ▼
Session Enum SQLi     Process   Archives   VM       Binary
   │    │    │          │         │       │           │
   └────┴────┘          └─────────┘       └───────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             ▼
                    Practical Security Skills
```

The foundational challenges introduce recognizable concepts first, while the more advanced reverse-engineering challenges require participants to combine multiple techniques.

This progression is valuable in a student-focused CTF because it encourages participants to build confidence with familiar tools before confronting more complex binaries.

---

# Coordinator Perspective

The broader challenge set was designed not only to test whether participants could recover flags, but also to expose them to realistic investigation workflows.

The Web challenges demonstrate how seemingly small implementation mistakes can become security vulnerabilities:

```text
Client-controlled state
        ↓
Authorization problem

Unlinked resource
        ↓
Enumeration problem

Unsafe SQL
        ↓
Authentication bypass

Frontend secret
        ↓
Information disclosure
```

The Linux challenges reinforce the importance of understanding the operating environment:

```text
Processes
   +
/proc
   +
Files
   +
Archives
   ↓
System-level investigation
```

The reverse-engineering challenges then build on that investigative mindset:

```text
Artifact
   ↓
Evidence
   ↓
Hypothesis
   ↓
Analysis
   ↓
Reconstruction
   ↓
Verification
```

This was the central design goal: make students **investigate the system rather than guess the answer**.

---

# Final Challenge-Set Takeaway

Across Web, Linux, File Investigation, Steganography, OSINT, and Reverse Engineering, the same fundamental methodology repeatedly appears:

> **Observe first. Understand the system. Identify the trust boundary. Recover the hidden transformation. Verify the result.**

That is ultimately what a good CTF should teach.

The flag is the reward, but the real outcome is the skill developed while finding it.


# Part II — The Blackout Protocol

## Challenge Profile

| Property | Details |
|---|---|
| Category | Reverse Engineering |
| Difficulty | Hard |
| Platform | Linux x86-64 |
| Primary Artifact | `blackout_protocol.zip` |
| Final Flag | `WHITESEC{th3_bl4ck0ut_w4s_n3v3r_0v3r}` |

### Technical Profile

| Attribute | Value |
|---|---|
| Category | Reverse Engineering |
| Difficulty | Hard |
| Architecture | Linux x86-64 |
| Primary artifact | `blackout_protocol.zip` |
| Core techniques | File carving, layered decoding, custom VM analysis, anti-debugging |
| Flag | `WHITESEC{th3_bl4ck0ut_w4s_n3v3r_0v3r}` |

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

# Part III — `reverse_me_harder`

## Challenge Profile

| Property | Details |
|---|---|
| Category | Reverse Engineering |
| Difficulty | Hard |
| Architecture | Linux x86-64 |
| Binary Type | PIE, dynamically linked, stripped ELF |
| Final Flag | `WHITESEC{b7f3a1}` |

### Technical Profile

| Attribute | Value |
|---|---|
| Category | Reverse Engineering |
| Difficulty | Hard |
| Architecture | Linux x86-64 |
| Binary | PIE, dynamically linked, stripped ELF |
| Core techniques | Runtime code decryption, control-flow flattening, SipHash-2-4, anti-debugging |
| Flag | `WHITESEC{b7f3a1}` |

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

# Part IV — Comparative Analysis

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

# Part V — Technical Lessons

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

# Part VI — Challenge-Author Perspective

### Design Principles

The challenges were developed around four principles:

1. **Layered discovery** — each stage should provide evidence for the next.
2. **Technical justification** — solvers should understand *why* a technique succeeds.
3. **Progressive reduction** — complexity should decrease as the underlying mechanism is understood.
4. **Verifiability** — a recovered solution should be independently testable against the challenge logic.


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

# Part VII — A Reusable Reverse-Engineering Methodology

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


---

## Editorial Note on the Appendix

The appendix preserves the supplied coordinator-level walkthroughs for the broader challenge set. Where the source material explicitly identified an incomplete or unresolved solve path, this article retains that qualification instead of presenting an unsupported conclusion as fact.

---

# Appendix — WHITESEC BLACKOUT 1.0 Challenge Solve Guide

The two reverse-engineering challenges above were the focus of this deep dive. For completeness, this appendix documents the other challenge solve paths from the organizer guide as well.

The original guide covers eleven challenges across **Misc, OSINT, and Steganography**, separating intended solving steps from final flags. fileciteturn19file0L1-L8

> **Post-event note:** This appendix reflects the challenge material and solve paths documented for the event. Where the organizer notes identified an incomplete or ambiguous path, that limitation is explicitly called out rather than silently "fixing" it.

## Misc

### 1. Welcome to WHITESEC

The player receives:

```text
welcome.txt
```

The intended path is to read the file, follow its reference to the WHITESEC Discord bot, and investigate the available bot commands:

```text
/welcome
/howtoplay
/rules
/categories
/scoreboard
/support
/hint
```

The bot also reacts to:

```text
blackout
```

and responds with:

```text
Signal detected.

You found something the bot wasn't supposed to reveal.
Look beyond the obvious response.
```

The intended concept is **interactive investigation rather than simply reading the supplied file**. The organizer notes also record that the then-current bot implementation loaded `CTF_FLAG` without actually exposing it, so this challenge did not have a fully deterministic flag-discovery path in that version. fileciteturn19file0L9-L66

---

### 2. The Strange Note

The supplied text is:

```text
dGgzX3N0cjRuZzNyX3RoM19uMHQz
```

The intended technique is Base64 decoding.

Using CyberChef:

```text
From Base64
```

produces:

```text
th3_str4ng3r_th3_n0t3
```

The documented concept is **Base64 decoding**. The organizer notes explicitly point out that this output is not itself a complete `WHITESEC{...}` flag, so an additional intentional stage would be required for a deterministic flag path. fileciteturn19file0L68-L110

---

### 3. Broken Signal

The player receives `signal.txt`:

```text
Key: 23

Data:
QF9eQ1JEUlRseSd+ZHJIdCN5SHR2ZWVuSGQmcHl2e2Rq
```

The intended workflow is:

```text
Base64
  ↓
XOR with decimal key 23
```

In CyberChef:

```text
From Base64
→ XOR
```

with the XOR key set to decimal:

```text
23
```

The result is:

```text
WHITESEC{n0ise_c4n_carry_s1gnals}
```

This challenge demonstrates how two simple transformations can be chained to create a small but effective decoding exercise. fileciteturn19file0L114-L176

---

### 4. The Last Piece

The supplied ZIP contains:

```text
fragment_01.txt
fragment_02.txt
fragment_03.txt
fragment_04.txt
```

The intended process is straightforward:

1. Extract the archive.
2. Open each fragment.
3. Use the numbered filenames to establish the correct order.
4. Concatenate the contents.

The final flag is:

```text
WHITESEC{puzzl3_p13c3s_f1t_t0g3th3r}
```

The intended concept is **file extraction, ordering, and reconstruction**. fileciteturn19file0L179-L236

---

# OSINT

## 5. The Digital Footprint

The player receives an image containing:

```text
DCG91422
April 18 2026
Kumaraguru College of Technology
Coimbatore
```

The most distinctive clue is:

```text
DCG91422
```

The intended investigation is:

```text
DCG91422
      ↓
Search the web
      ↓
Cross-reference the event information
      ↓
Identify the relevant DCG event
```

The documented flag is:

```text
WHITESEC{ctf_dcg91422}
```

The intended skills are **search-engine investigation and cross-referencing public information**. fileciteturn19file0L238-L288

---

## 6. Lost in the Metadata

The supplied file is:

```text
lost_in_metadata.jpg
```

The challenge encourages the player to inspect metadata.

A standard first step is:

```bash
exiftool lost_in_metadata.jpg
```

The important EXIF clue is:

```text
The trail continues beyond the official page. Check its social media and coordinators.
```

This points toward:

```text
social media
+
coordinators
```

However, the organizer notes identify the second-stage public trail and final flag as not fully established in the documented version. That ambiguity is preserved here rather than inventing a missing step. fileciteturn19file0L291-L347

---

## 7. The Vanishing Profile

The player receives an X.com profile screenshot.

The intended workflow is:

```text
Profile screenshot
       ↓
Identify the relevant identity
       ↓
Search across platforms
       ↓
Find the corresponding LinkedIn profile
       ↓
Inspect profile details
```

The key detail is:

```text
Joined LinkedIn — September 2024
```

The documented flag is:

```text
WHITESEC{l1nk3d1n_s3pt3mb3r_2024}
```

The challenge demonstrates **identity correlation and cross-platform OSINT**.

For a public CTF, the organizer notes recommend using a fictional or organizer-controlled identity rather than information belonging to an unrelated real person. fileciteturn19file0L349-L395

---

## 8. The Final Trail

The intended investigation starts with:

```text
Panimalar Engineering College
CSE Department
```

The player locates:

```text
CSE Department BLOCK-1
```

and retrieves the map coordinates:

```text
13.049603080186209, 80.07507293716917
```

The documented flag is:

```text
WHITESEC{13.049603080186209_80.07507293716917}
```

The intended concept is **geolocation / Google Maps OSINT**. fileciteturn19file0L397-L438

---

# Steganography

## 9. A Picture Says More

The player receives:

```text
picture.jpg
```

Nothing suspicious needs to be visible in the image itself.

A useful first check is:

```bash
strings picture.jpg
```

Search for:

```text
WHITESEC
```

The hidden data contains:

```text
[WHITESEC ARCHIVE]
flag=WHITESEC{p1ctur3s_s4y_m0r3}
```

The final flag is:

```text
WHITESEC{p1ctur3s_s4y_m0r3}
```

The intended concept is **hidden text inside an image**. fileciteturn19file0L440-L497

---

## 10. Pixels Don't Lie

The player receives:

```text
behind_the_pixels.png
```

The clue about the "smallest changes" points toward pixel-level steganography.

A useful tool is:

```bash
zsteg -a behind_the_pixels.png
```

The relevant result is:

```text
b1,rgb,lsb,xy
```

This indicates:

```text
1 bit
RGB channels
LSB
XY order
```

The extracted data reveals:

```text
WHITESEC{b3h1nd_th3_p1x3ls}
```

The intended technique is **RGB Least Significant Bit steganography**. fileciteturn19file0L499-L560

---

## 11. The Invisible Layer

The supplied image is:

```text
hidden_layer.png
```

The challenge description points toward an invisible component:

```text
The image has more than one layer.

What appears to be empty
may not be empty at all.

Look beyond what is visible.
```

This suggests investigating transparency and the PNG alpha channel.

The flag is encoded into the **least significant bits of the alpha channel**.

Conceptually:

```text
RGBA
 │
 └── Alpha channel
        │
        ▼
       LSBs
        │
        ▼
       Flag
```

The documented flag is:

```text
WHITESEC{th3_h1dd3n_l4y3r}
```

The organizer notes also suggest that this challenge could be made more beginner-friendly by providing a clearer extraction path using common tools. fileciteturn19file0L562-L613

---

# Complete Flag Reference

| # | Category | Challenge | Flag / Status |
|---:|---|---|---|
| 1 | Misc | Welcome to WHITESEC | Path incomplete in documented version |
| 2 | Misc | The Strange Note | Second stage required in documented version |
| 3 | Misc | Broken Signal | `WHITESEC{n0ise_c4n_carry_s1gnals}` |
| 4 | Misc | The Last Piece | `WHITESEC{puzzl3_p13c3s_f1t_t0g3th3r}` |
| 5 | OSINT | The Digital Footprint | `WHITESEC{ctf_dcg91422}` |
| 6 | OSINT | Lost in the Metadata | Second stage not established in documented version |
| 7 | OSINT | The Vanishing Profile | `WHITESEC{l1nk3d1n_s3pt3mb3r_2024}` |
| 8 | OSINT | The Final Trail | `WHITESEC{13.049603080186209_80.07507293716917}` |
| 9 | Steganography | A Picture Says More | `WHITESEC{p1ctur3s_s4y_m0r3}` |
| 10 | Steganography | Pixels Don't Lie | `WHITESEC{b3h1nd_th3_p1x3ls}` |
| 11 | Steganography | The Invisible Layer | `WHITESEC{th3_h1dd3n_l4y3r}` |

The organizer guide explicitly identifies three documented challenges as needing completion before release: **Welcome to WHITESEC**, **The Strange Note**, and **Lost in the Metadata**. fileciteturn19file0L615-L639

---

## Closing Note

The broader WHITESEC BLACKOUT 1.0 challenge set demonstrates that CTF solving is not limited to exploit development or binary reversing.

The event also required participants to move between:

```text
Misc
  ├── Interactive investigation
  ├── Encoding
  └── File reconstruction

OSINT
  ├── Search
  ├── Identity correlation
  └── Geolocation

Steganography
  ├── Hidden file data
  ├── Pixel LSBs
  └── Alpha-channel analysis

Reverse Engineering
  ├── File-format recovery
  ├── Custom VM analysis
  ├── Runtime code decryption
  ├── Anti-debugging
  ├── Cryptographic validation
  └── Controlled brute force
```

That variety is what makes a CTF valuable as a learning environment: the solver is repeatedly forced to ask **what kind of problem am I actually looking at?**


# Part VIII — Final Takeaways

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


---

## Author

**Hemanth Karal Varmaa M**  
*WHITESEC Cybersecurity Club · Challenge Author / Coordinator*

**WHITESEC BLACKOUT 1.0**  
*Hack • Learn • Compete*

