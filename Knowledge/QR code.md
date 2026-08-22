---
title: QR Code
tags: [knowledge, encoding, qr]
---

# 📷 QR Code – How It Works

**Quick Response Code** — a 2D matrix barcode invented in 1994 by Masahiro Hara at Denso Wave (Japan), originally for tracking automobile parts. Unlike a 1D barcode that encodes data in horizontal lines, a QR code encodes data in a 2D grid of black and white squares.

---

## 🧱 Structure

A QR code is divided into functional regions:

| Region | Purpose |
|--------|---------|
| **Finder Patterns** | Three square "eyes" in three corners — used to detect position and orientation |
| **Alignment Patterns** | Smaller squares in larger QR versions — corrects distortion |
| **Timing Patterns** | Alternating black/white strips — tells the scanner the module grid size |
| **Format Information** | Stores error correction level and mask pattern |
| **Data + ECC Region** | The actual encoded payload + redundancy bits |
| **Quiet Zone** | White border around the entire code — prevents false reads |

---

## ⚙️ How Encoding Works

### Step 1 — Choose data mode
QR supports multiple encoding modes depending on content:
- **Numeric** — digits only (most compact, 3.3 bits/char)
- **Alphanumeric** — uppercase + some symbols (5.5 bits/char)
- **Byte** — UTF-8 / binary (8 bits/char)
- **Kanji** — Japanese characters (13 bits/char)

### Step 2 — Apply error correction
QR uses **Reed-Solomon error correction**, which adds redundant data so the code can still be read even if partially damaged or obscured.

| Level | Can recover up to |
|-------|------------------|
| **L** (Low) | 7% of codewords |
| **M** (Medium) | 15% |
| **Q** (Quartile) | 25% |
| **H** (High) | 30% |

This is why logos can be placed in the center of a QR code — the destroyed data is reconstructed via ECC.

### Step 3 — Arrange into matrix
Data bits are placed in the grid following a specific zigzag path (from bottom-right, going up). A **mask pattern** is then XOR'd over the data to avoid large uniform areas that could confuse scanners. 8 mask patterns exist; the best one is chosen by scoring.

### Step 4 — Add metadata
Format strips encode the chosen error correction level + mask pattern. Version strips (for large QRs v7+) encode the QR version.

---

## 📐 Versions and Capacity

QR codes come in **40 versions** (1–40). Each version increases the grid by 4 modules on each side:
- Version 1: 21×21 modules
- Version 40: 177×177 modules

| Version | Max characters (byte mode, L) |
|---------|-------------------------------|
| 1 | 17 |
| 10 | 271 |
| 40 | 2,953 |

---

## 🔍 How a Scanner Reads It

1. **Detect** the three finder patterns and determine rotation/skew
2. **Sample** the module grid (each cell is read as black=1 or white=0)
3. **Strip** the mask XOR
4. **Decode** Reed-Solomon and reconstruct any damaged blocks
5. **Parse** the data payload based on mode indicators

A standard smartphone camera does all of this in real time at ~30 fps.

---

## 🔗 Relacionado
- [[Knowledge/Abstraction|Abstraction]]
- [[CS/CS|CS]] — encoding algorithms

## ↩️ Navegación
- [[Knowledge/Knowledge|🧠 Knowledge]] → [[HOME|🏠 HOME]]
