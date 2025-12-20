# Audio Fingerprinting Technology
## How We Detect Duplicate & Similar Music

> **For Music Industry Professionals**
>
> This document explains how our audio fingerprinting technology identifies duplicate and similar audio files in your music catalog.

---

## What Is Audio Fingerprinting?

Audio fingerprinting creates a **unique "DNA profile"** for each song by analyzing its frequency content. Just as human fingerprints identify a person regardless of clothing, audio fingerprints identify a song regardless of:

- **File format** (MP3, M4A, WAV, FLAC, AAC, AIFF)
- **Audio quality** (128 kbps vs 320 kbps)
- **Slight edits** (trimmed intros, fade-outs, radio edits)

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     AUDIO FINGERPRINTING PIPELINE                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Audio File                                                         │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 1. Audio Decoding    │  ← Converts any format to raw waveform   │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 2. Mono Conversion   │  ← Mix stereo channels for consistency   │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 3. Windowing         │  ← 2048 samples/window, 50% overlap       │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 4. FFT Analysis      │  ← Frequency spectrum extraction         │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 5. Peak Extraction   │  ← Top 5 frequencies per window          │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 6. Hash Generation   │  ← Condensed signature for fast lookup   │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   AudioFingerprint (peaks + hash + duration)                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Process

### Step 1: Audio Decoding

All audio formats are decoded to raw PCM waveform data. This ensures **format-independent analysis** — an MP3 and WAV of the same song will produce identical waveforms.

### Step 2: Mono Conversion

Stereo files are mixed down to mono by averaging left and right channels. This ensures consistent fingerprints regardless of stereo width or panning differences.

### Step 3: Sliding Window Analysis

The audio is analyzed in small overlapping segments:

```
Audio Waveform:   |───────────────────────────────────────────→
                   ↑        ↑        ↑        ↑        ↑
Window 1:         [====2048====]
Window 2:              [====2048====]     ← 50% overlap (1024 samples)
Window 3:                   [====2048====]
Window 4:                        [====2048====]
...
```

**Parameters:**
| Setting | Value | Description |
|---------|-------|-------------|
| Window Size | 2048 samples | ~46 milliseconds at 44.1kHz |
| Overlap | 50% (1024 samples) | Ensures no audio is missed |
| Windows per 30s | ~1,290 | High temporal resolution |

Each window is smoothed with a **Hann window function** to prevent edge artifacts:

```
Hann Window Shape:
1.0 ─────────────╱╲─────────────
                ╱  ╲
               ╱    ╲
              ╱      ╲
0.0 ─────────╱        ╲─────────
             0       2048
```

### Step 4: FFT Spectral Analysis

Each window is transformed from time-domain (waveform) to frequency-domain (spectrum) using **Fast Fourier Transform (FFT)**.

This reveals which frequencies are present at each moment:

```
Time Domain (Waveform):          Frequency Domain (Spectrum):
                                 
    ╱╲  ╱╲  ╱╲                        ▂▅█▃
   ╱  ╲╱  ╲╱  ╲      ──FFT──→      ▁▃█████▂▁
  ╱              ╲                  └───────────→
                                    0Hz    3kHz
```

**Frequency Resolution:**
- At 44.1kHz sample rate with 2048-sample FFT: **~21.5 Hz per bin**
- Analysis range: **300 Hz - 3000 Hz** (vocal and instrument fundamentals)

### Step 5: Peak Extraction

For each window, the **5 strongest frequencies** (local maxima) are extracted:

```
Frequency Spectrum for One Window:

Magnitude
    │      ▲ Peak 2
    │   ▲  │     ▲ Peak 4
    │   │  │  ▲  │
    │ ▲ │  │  │  │    ▲ Peak 5
    │ │ │  │  │  │ ▲  │
────┴─┴─┴──┴──┴──┴─┴──┴────────→ Frequency (Hz)
   300   600  900 1200 1500 ... 3000

Output: [345 Hz, 620 Hz, 890 Hz, 1240 Hz, 2100 Hz]
```

**Why 300-3000 Hz?**
- Below 300 Hz: Bass/rumble, too variable across encodings
- Above 3000 Hz: Highs lost in lossy compression
- 300-3000 Hz: Vocals, lead instruments — the "signature" of the song

### Step 6: Hash Generation

A condensed hash is created for fast pre-filtering:

1. Sample ~10 windows evenly across the fingerprint
2. Take the top peak frequency from each
3. Quantize to 50 Hz bands
4. Convert to hexadecimal string

**Example:**
```
Top peaks from 10 windows: [350, 400, 380, 420, 360, 390, 410, 370, 385, 395]
Quantized (÷50):          [7,   8,   7,   8,   7,   7,   8,   7,   7,   7  ]
Hash:                     "07080708070708070707"
```

---

## Fingerprint Comparison

### Comparison Pipeline

```
┌───────────────────────────────────────────────────────────────┐
│                    COMPARISON PIPELINE                         │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│   Fingerprint A              Fingerprint B                     │
│        ↓                          ↓                            │
│   ┌───────────────────────────────────────────────────────┐   │
│   │ 1. Direct Comparison (offset = 0)                      │   │
│   │    → If similarity ≥ 85%, return immediately           │   │
│   └───────────────────────────────────────────────────────┘   │
│        ↓ (if < 85%)                                           │
│   ┌───────────────────────────────────────────────────────┐   │
│   │ 2. Sliding Window Comparison                           │   │
│   │    → Try offsets: 20, 40, 60, ... 400 windows         │   │
│   │    → Check both directions (A offset, B offset)        │   │
│   │    → Early exit if ≥ 85% found                         │   │
│   └───────────────────────────────────────────────────────┘   │
│        ↓                                                       │
│   Return best similarity found (0.0 - 1.0)                     │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

### Handling Trimmed/Offset Audio

Same song may be edited differently:

```
Song A: [INTRO------|=====ACTUAL SONG======================]
Song B: [=====ACTUAL SONG======================]
         ↑
         Offset Song A by ~130 windows to align
```

**Parameters:**
| Setting | Value | Description |
|---------|-------|-------------|
| Max Offset | 400 windows | ~10 seconds at 44.1kHz |
| Step Size | 20 windows | ~0.5 seconds between checks |
| Tolerance | ±50 Hz | Accounts for encoding variations |

### Window-by-Window Matching

For each sampled window pair:
1. Compare the 5 peak frequencies
2. A frequency "matches" if within ±50 Hz
3. Window matches if ≥50% of peaks match
4. Overall similarity = (matching windows) / (total windows sampled)

---

## Duplicate Detection Engine

### Two-Phase Approach

```
┌─────────────────────────────────────────────────────────────────┐
│                    DUPLICATE ENGINE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   All Tracks with Fingerprints                                   │
│        ↓                                                         │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ PHASE 1: Hash Bucketing                                 │    │
│   │   Group tracks by identical hash                        │    │
│   │   Only compare within same bucket (very fast)           │    │
│   └────────────────────────────────────────────────────────┘    │
│        ↓                                                         │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ PHASE 2: Cross-Hash Comparison                          │    │
│   │   Compare remaining tracks against ALL tracks           │    │
│   │   Catches similar fingerprints in different buckets     │    │
│   └────────────────────────────────────────────────────────┘    │
│        ↓                                                         │
│   Duplicate Groups                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Phase 1** provides fast O(N) grouping before expensive O(N²) comparison.

**Phase 2** catches edge cases where hash quantization put similar tracks in different buckets.

---

## Understanding Match Thresholds

The **Match Threshold** determines detection sensitivity:

| Threshold | Matches Found | Best For |
|-----------|---------------|----------|
| **90-100%** | Only exact duplicates | Finding true duplicates with different quality |
| **80-90%** | Same song, minor differences | Radio edits, slightly trimmed versions |
| **70-80%** | Same song, moderate differences | Different intros/outros, extended versions |
| **60-70%** | Similar sounding content | Remixes, faithful cover versions |
| **Below 60%** | Loosely similar | May produce false positives |

### What Each Threshold Catches

| Threshold | Catches |
|-----------|---------|
| **90-100%** | Same song, different formats/bitrates, renamed files |
| **80-90%** | + Radio edits, remastered versions |
| **70-80%** | + Extended mixes, different fade-in/out |
| **60-70%** | + Remixes (same core), very faithful covers |

### What It CANNOT Match (Any Threshold)

❌ Different songs in the same genre
❌ Songs by same artist with different melodies
❌ Songs that "feel" similar but have different notes

**Why?** Fingerprints match on **actual frequencies played**, not subjective qualities.

---

## Accuracy Expectations

| Scenario | Expected Match Rate |
|----------|---------------------|
| Same song, different MP3 quality | 95-100% |
| Same song, MP3 vs M4A | 90-98% |
| Album version vs Radio edit | 80-95% |
| Original vs Extended mix | 75-90% |
| Original vs Acoustic remix | 55-75% |
| Original vs Very different remix | 30-50% |
| Completely different songs | 10-35% |

---

## Configuration Parameters

| Parameter | Value | Impact |
|-----------|-------|--------|
| Sample Duration | 10s, 30s, 60s, Full | Accuracy vs Speed |
| Similarity Threshold | 50% - 100% | Strictness of matching |
| FFT Size | 2048 samples | Frequency resolution |
| Overlap | 50% | Time resolution |
| Peaks per Window | 5 | Fingerprint precision |
| Frequency Range | 300-3000 Hz | Focus on vocals/instruments |
| Frequency Tolerance | ±50 Hz | Match flexibility |
| Max Offset | ~10 seconds | Trim tolerance |

---

## Performance

- **Parallel Processing**: Multiple files analyzed concurrently
- **Hash Pre-Filtering**: O(N) grouping before O(N²) comparison
- **Sampled Comparison**: Every 10th window (10x speedup)
- **Early Exit**: Stops on high-confidence matches
- **Hardware Acceleration**: Uses Apple's Accelerate framework for FFT

Typical performance: **Thousands of files in minutes**

---

## Business Applications

### 1. Catalog Deduplication
**Recommended:** 85-95% threshold
- Catches format/quality duplicates
- Low false positive rate

### 2. Version Grouping
**Recommended:** 70-85% threshold
- Groups related versions together
- Helps with rights management

### 3. Cover/Remix Detection (Experimental)
**Recommended:** 60-75% threshold
- May find faithful covers
- Manual verification required

---

## Summary

Our fingerprinting technology analyzes the **spectral DNA** of audio:

1. Decode to format-independent waveform
2. Analyze in overlapping windows with FFT
3. Extract top 5 peak frequencies per window
4. Generate hash for fast pre-filtering
5. Compare with sliding window alignment

This enables **format-agnostic, quality-agnostic, edit-tolerant** duplicate detection with adjustable sensitivity.

---

*For implementation details, see the developer documentation: `AUDIO_FINGERPRINTING.md`*
