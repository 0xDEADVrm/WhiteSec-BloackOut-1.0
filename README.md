# 🏴 WHITESEC BLACKOUT 1.0 — Challenge Walkthrough

Official organizer solution guide for all 11 challenges. Each entry lists the participant-facing materials, the intended solve path, the concept being tested, and the final flag. Three challenges are flagged **⚠️ Incomplete** and need organizer action before release — see the [summary table](#-flag-summary) and [outstanding items](#-outstanding-items-before-release) at the end.

## Table of Contents

- [Misc](#-misc)
  - [1. Welcome to WHITESEC](#1-welcome-to-whitesec--100-pts)
  - [2. The Strange Note](#2-the-strange-note--100-pts)
  - [3. Broken Signal](#3-broken-signal--125-pts)
  - [4. The Last Piece](#4-the-last-piece--150-pts)
- [OSINT](#-osint)
  - [5. The Digital Footprint](#5-the-digital-footprint--75-pts)
  - [6. Lost in the Metadata](#6-lost-in-the-metadata--100-pts)
  - [7. The Vanishing Profile](#7-the-vanishing-profile--125-pts)
  - [8. The Final Trail](#8-the-final-trail--150-pts)
- [Steganography](#️-steganography)
  - [9. A Picture Says More](#9-a-picture-says-more--50-pts)
  - [10. Pixels Don't Lie](#10-pixels-dont-lie--75-pts)
  - [11. The Invisible Layer](#11-the-invisible-layer--100-pts)
- [Flag Summary](#-flag-summary)
- [Outstanding Items Before Release](#-outstanding-items-before-release)

---

## 🧩 Misc

### 1. Welcome to WHITESEC — 100 pts

**Receives:** `welcome.txt`

**Solve:**
1. Open `welcome.txt` and read the visible message.
2. The message points toward the WHITESEC Discord bot.
3. Join the official WHITESEC BLACKOUT Discord server.
4. Try the bot commands: `/welcome`, `/howtoplay`, `/rules`, `/categories`, `/scoreboard`, `/support`, `/hint`.
5. The bot also reacts to the keyword `blackout`, returning:
   ```
   Signal detected.

   You found something the bot wasn't supposed to reveal.
   Look beyond the obvious response.
   ```

**Concept:** Investigating bot behavior beyond the obvious file/response.

> ⚠️ **Incomplete** — the bot currently loads `CTF_FLAG` but never reveals it. There is no complete flag-discovery path. Add an intentional hidden clue/flag-reveal mechanism before publishing.

---

### 2. The Strange Note — 100 pts

**Receives:** a text file containing:
```
dGgzX3N0cjRuZzNyX3RoM19uMHQz
```

**Solve:**
1. Recognize the string as Base64.
2. Decode with CyberChef (`From Base64`).

**Result:**
```
th3_str4ng3r_th3_n0t3
```

**Concept:** Base64 decoding.

> ⚠️ **Incomplete** — the decoded value is not itself a valid `WHITESEC{...}` flag. Add a follow-up step so decoding resolves to a complete flag.

---

### 3. Broken Signal — 125 pts

**Receives:** `signal.txt`
```
Key: 23

Data:
QF9eQ1JEUlRseSd+ZHJIdCN5SHR2ZWVuSGQmcHl2e2Rq
```

**Solve:**
1. Open CyberChef.
2. Add `From Base64`.
3. Add `XOR`.
4. Set the XOR key to `23` (decimal).

**Flag:**
```
WHITESEC{n0ise_c4n_carry_s1gnals}
```

**Concept:** Base64 + XOR.

---

### 4. The Last Piece — 150 pts

**Receives:** a ZIP archive containing:
```
fragment_01.txt
fragment_02.txt
fragment_03.txt
fragment_04.txt
```

**Solve:**
1. Extract the ZIP.
2. Open each fragment — every file holds part of the final message.
3. Use the filenames to determine order.
4. Concatenate `fragment_01` → `fragment_04`.

**Flag:**
```
WHITESEC{puzzl3_p13c3s_f1t_t0g3th3r}
```

**Concept:** File extraction, ordering, and reconstruction.

---

## 🕵️ OSINT

### 5. The Digital Footprint — 75 pts

**Receives:** a conference poster/image containing:
```
DCG91422
April 18 2026
Kumaraguru College of Technology
Coimbatore
```

**Solve:**
1. Review the visible information.
2. Search the web for the unusual identifier `DCG91422`.
3. Cross-reference the result with the conference details shown.
4. Combine the identifier with the expected flag format.

**Flag:**
```
WHITESEC{ctf_dcg91422}
```

**Concept:** Search-engine investigation + cross-referencing public information.

---

### 6. Lost in the Metadata — 100 pts

**Receives:** `lost_in_metadata.jpg`

**Hint:** "Not everything important can be seen."

**Solve:**
1. Run:
   ```bash
   exiftool lost_in_metadata.jpg
   ```
   (or use an online metadata viewer.)
2. Inspect the EXIF fields. The relevant one reads:
   ```
   The trail continues beyond the official page. Check its social media and coordinators.
   ```
3. Follow the clue to the intended social-media/coordinator trail.

**Concept:** EXIF metadata analysis + social-media OSINT.

> ⚠️ **Incomplete** — the EXIF clue points to "social media" and "coordinators," but the second-stage public trail and final flag have not been established. Do not publish as complete until this trail is built.

---

### 7. The Vanishing Profile — 125 pts

**Receives:** an X.com (Twitter) profile screenshot.

**Solve:**
1. Examine the visible profile info — the answer isn't there directly.
2. Identify the relevant username/identity.
3. Search for the same identity on other platforms.
4. Locate the corresponding LinkedIn profile.
5. Inspect the LinkedIn details:
   ```
   Joined LinkedIn — September 2024
   ```

**Flag:**
```
WHITESEC{l1nk3d1n_s3pt3mb3r_2024}
```

**Concept:** Identity correlation + cross-platform OSINT.

> 💡 **Recommendation:** use a fictional or organizer-controlled identity for public competitions rather than an unrelated real person's information.

---

### 8. The Final Trail — 150 pts

**Receives:** a description pointing to a physical location.

**Solve:**
1. Search Google Maps for **Panimalar Engineering College, CSE Department**.
2. Locate **CSE Department, Block-1**.
3. Right-click the pin and copy the coordinates:
   ```
   13.049603080186209, 80.07507293716917
   ```

**Flag:**
```
WHITESEC{13.049603080186209_80.07507293716917}
```

**Concept:** Geolocation / Google Maps OSINT.

---

## 🖼️ Steganography

### 9. A Picture Says More — 50 pts

**Receives:** `picture.jpg`

**Solve:**
1. Open the image — nothing suspicious is visible.
2. Inspect embedded strings:
   ```bash
   strings picture.jpg      # Linux
   strings picture.jpg      # Windows PowerShell (or use a hex editor)
   ```
3. Search the output for `WHITESEC`:
   ```
   [WHITESEC ARCHIVE]
   flag=WHITESEC{p1ctur3s_s4y_m0r3}
   ```

**Flag:**
```
WHITESEC{p1ctur3s_s4y_m0r3}
```

**Concept:** Hidden text embedded inside an image file.

---

### 10. Pixels Don't Lie — 75 pts

**Receives:** `behind_the_pixels.png`

**Hint:** "Smallest changes carry the biggest secrets."

**Solve:**
1. Run `zsteg`:
   ```bash
   zsteg -a behind_the_pixels.png
   ```
2. Look for a readable payload — the relevant detection is `b1,rgb,lsb,xy` (1 bit, RGB channels, least-significant bit, XY order).
3. Decode to reveal the flag.

**Flag:**
```
WHITESEC{b3h1nd_th3_p1x3ls}
```

**Concept:** RGB least-significant-bit (LSB) steganography.

---

### 11. The Invisible Layer — 100 pts

**Receives:** `hidden_layer.png`

**Hint:**
```
The image has more than one layer.
What appears to be empty may not be empty at all.
Look beyond what is visible.
```

**Solve:**
1. Recognize this points to transparency / alpha-channel data.
2. Extract the RGBA alpha channel (e.g., with a short Python script using Pillow).
3. Recover the least-significant bits of the alpha channel to reveal the message.

**Flag:**
```
WHITESEC{th3_h1dd3n_l4y3r}
```

**Concept:** Alpha-channel LSB steganography.

> 💡 **Note:** technically solvable, but harder than Steg #1/#2 for a beginner-level CTF — the extraction isn't as approachable with common beginner tools. Consider simplifying before release.

---

## 📋 Flag Summary

| # | Category | Challenge | Points | Flag / Status |
|---|----------|-----------|--------|----------------|
| 1 | Misc | Welcome to WHITESEC | 100 | ⚠️ Needs flag-discovery path |
| 2 | Misc | The Strange Note | 100 | ⚠️ Needs second decoding step |
| 3 | Misc | Broken Signal | 125 | `WHITESEC{n0ise_c4n_carry_s1gnals}` |
| 4 | Misc | The Last Piece | 150 | `WHITESEC{puzzl3_p13c3s_f1t_t0g3th3r}` |
| 5 | OSINT | The Digital Footprint | 75 | `WHITESEC{ctf_dcg91422}` |
| 6 | OSINT | Lost in the Metadata | 100 | ⚠️ Needs second-stage trail |
| 7 | OSINT | The Vanishing Profile | 125 | `WHITESEC{l1nk3d1n_s3pt3mb3r_2024}` |
| 8 | OSINT | The Final Trail | 150 | `WHITESEC{13.049603080186209_80.07507293716917}` |
| 9 | Steganography | A Picture Says More | 50 | `WHITESEC{p1ctur3s_s4y_m0r3}` |
| 10 | Steganography | Pixels Don't Lie | 75 | `WHITESEC{b3h1nd_th3_p1x3ls}` |
| 11 | Steganography | The Invisible Layer | 100 | `WHITESEC{th3_h1dd3n_l4y3r}` |

---

## 🔴 Outstanding Items Before Release

- [ ] **Welcome to WHITESEC** — Discord bot loads `CTF_FLAG` but never reveals it; add a genuine discovery mechanism.
- [ ] **The Strange Note** — Base64 decoding yields `th3_str4ng3r_th3_n0t3`, not a complete flag; add a follow-up step.
- [ ] **Lost in the Metadata** — EXIF clue references a social-media/coordinator trail that hasn't been built out to a final flag.

**Recommendation:** resolve all three items above before announcing WHITESEC BLACKOUT 1.0, so every challenge has a fully deterministic solve path and no participant is forced to guess.
