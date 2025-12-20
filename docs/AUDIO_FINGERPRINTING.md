# Custom Audio Fingerprinting Algorithm for Duplicate Detection

> **Technical Deep-Dive Document**
> 
> This document explains the custom FFT-based audio fingerprinting algorithm used in the Duplicate Music Finder app to detect duplicate audio files regardless of file format, bitrate, or metadata differences.

---

## Overview

The algorithm analyzes the **spectral characteristics** of audio files using Fast Fourier Transform (FFT) to create a "fingerprint" that can identify the same song even when:
- Encoded at different bitrates (128kbps vs 320kbps)
- In different formats (MP3 vs M4A vs WAV)
- Slightly trimmed or offset (up to ~10 seconds difference)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     AUDIO FINGERPRINTING PIPELINE                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Audio File                                                         │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 1. Audio Decoding    │  ← AVAudioFile (any format → PCM)        │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 2. Mono Conversion   │  ← Mix stereo channels                    │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 3. Windowing         │  ← 2048 samples/window, 50% overlap       │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 4. FFT Analysis      │  ← Apple Accelerate/vDSP framework        │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 5. Peak Extraction   │  ← Top 5 frequencies per window          │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   ┌──────────────────────┐                                           │
│   │ 6. Hash Generation   │  ← Condensed hash for fast filtering     │
│   └──────────────────────┘                                           │
│       ↓                                                              │
│   AudioFingerprint (peaks + hash + duration)                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Audio Decoding

**Framework:** `AVFoundation` (AVAudioFile)

```swift
let audioFile = try AVAudioFile(forReading: url)
let sampleRate = Float(audioFile.processingFormat.sampleRate)  // Usually 44100 Hz
```

**What happens:**
- Any audio format (MP3, M4A, AAC, FLAC, WAV, AIFF) is decoded to raw PCM samples
- This makes the fingerprint **format-independent** - the same song sounds identical regardless of container

---

## Step 2: Sample Selection

**Configuration Options:**
| Duration | Frames at 44.1kHz | Use Case |
|----------|-------------------|----------|
| 10 seconds | 441,000 | Fast scanning |
| 30 seconds | 1,323,000 | Balance speed/accuracy |
| 60 seconds | 2,646,000 | High accuracy |
| Full Track | All frames | Maximum accuracy |

**Skip Intro:** The algorithm skips the first **5 seconds** to avoid:
- Silence at the start
- Intro jingles or effects
- Different fade-in styles

```swift
let skipSeconds: Double = 5.0
let skipFrames = AVAudioFramePosition(skipSeconds * Double(sampleRate))
let startFrame = min(skipFrames, Int64(totalFrames) - Int64(targetFrames))
```

---

## Step 3: Mono Conversion

**Why:** Stereo files have 2 channels with potentially different data. To ensure consistent fingerprints regardless of stereo/mono source, we mix down to mono.

```swift
// Mix channels to mono
var monoSamples = [Float](repeating: 0, count: frameCount)
for channel in 0..<channelCount {
    let samples = UnsafeBufferPointer(start: channelData[channel], count: frameCount)
    for i in 0..<frameCount {
        monoSamples[i] += samples[i] / Float(channelCount)
    }
}
```

**Result:** Single array of `Float` values representing the audio waveform.

---

## Step 4: Windowing (Sliding Window Analysis)

**The Core Concept:**

Audio is analyzed in small overlapping chunks called "windows":

```
Audio Stream:     |-----------------------------------------------→
                   ↑        ↑        ↑        ↑        ↑
Window 1:         [====2048====]
Window 2:              [====2048====]     ← 50% overlap (1024 samples)
Window 3:                   [====2048====]
Window 4:                        [====2048====]
...
```

**Parameters:**
| Parameter | Value | Explanation |
|-----------|-------|-------------|
| `fftSize` | 2048 samples | ~46ms at 44.1kHz - captures enough frequency detail |
| `hopSize` | 1024 samples | 50% overlap - ensures no audio is missed |

**Window Count Example:**
- 30-second clip at 44.1kHz = 1,323,000 samples
- Windows = (1,323,000 - 2048) / 1024 ≈ **1,290 windows**

### Hann Window Function

Before FFT, each window is multiplied by a **Hann window** to reduce spectral leakage:

```swift
var window = [Float](repeating: 0, count: fftSize)
vDSP_hann_window(&window, vDSP_Length(fftSize), Int32(vDSP_HANN_NORM))
vDSP_vmul(windowSamples, 1, window, 1, &windowedSamples, 1, vDSP_Length(fftSize))
```

**Visual of Hann Window:**
```
1.0 ─────────────╱╲─────────────
                ╱  ╲
               ╱    ╲
              ╱      ╲
0.0 ─────────╱        ╲─────────
             0       2048
```

This prevents "edge effects" where frequencies appear stronger than they are due to the window boundary.

---

## Step 5: FFT (Fast Fourier Transform)

**Framework:** Apple's `Accelerate` framework (`vDSP`)

**What FFT Does:** Converts time-domain audio (waveform) to frequency-domain (spectrum).

```swift
let log2n = vDSP_Length(log2(Float(fftSize)))  // = 11 for 2048
let fftSetup = vDSP_create_fftsetup(log2n, FFTRadix(kFFTRadix2))

vDSP_fft_zrip(fftSetup, &splitComplex, 1, log2n, FFTDirection(FFT_FORWARD))
```

**Output:** Array of 1024 complex numbers representing frequency magnitudes from 0 Hz to 22,050 Hz (Nyquist frequency).

### Frequency Resolution

```
Bin Width = Sample Rate / FFT Size
          = 44100 / 2048
          ≈ 21.5 Hz per bin
```

| Bin Index | Frequency Range |
|-----------|-----------------|
| 0 | 0 - 21.5 Hz |
| 1 | 21.5 - 43 Hz |
| ... | ... |
| 14 | 300 - 322 Hz |
| ... | ... |
| 139 | 2980 - 3000 Hz |

### Magnitude Calculation

```swift
// Calculate magnitude = sqrt(real² + imag²)
vDSP_zvmags(&splitComplex, 1, &magnitudes, 1, vDSP_Length(fftSize / 2))
```

---

## Step 6: Peak Extraction

**Goal:** Find the strongest frequencies in each window (these characterize the sound).

**Frequency Range Filtering:**
| Parameter | Value | Reason |
|-----------|-------|--------|
| `minFrequency` | 300 Hz | Ignore sub-bass rumble/noise |
| `maxFrequency` | 3000 Hz | Focus on vocal/instrument fundamentals |

**Peak Detection Algorithm:**

```swift
let binWidth = sampleRate / Float(fftSize)  // ~21.5 Hz
let minBin = Int(300 / binWidth)   // ≈ 14
let maxBin = Int(3000 / binWidth)  // ≈ 139

// Find local maxima
for i in (minBin + 1)..<maxBin {
    if magnitudes[i] > magnitudes[i - 1] && magnitudes[i] > magnitudes[i + 1] {
        // This is a peak!
        let frequency = Float(i) * binWidth
        peaks.append((frequency, magnitudes[i]))
    }
}

// Sort by magnitude, take top 5
peaks.sort { $0.magnitude > $1.magnitude }
let topPeaks = peaks.prefix(5)
```

**Output per window:** Array of 5 frequencies (sorted ascending), e.g., `[345.0, 520.0, 890.0, 1240.0, 2100.0]`

---

## Step 7: Hash Generation

**Purpose:** Create a condensed "signature" for fast pre-filtering before detailed comparison.

```swift
static func generateHash(from peaks: [[Float]]) -> String {
    var hashComponents: [UInt32] = []
    
    // Sample ~10 windows evenly distributed
    let step = max(1, peaks.count / 10)
    
    for i in stride(from: 0, to: peaks.count, by: step) {
        if let topPeak = peaks[i].first {
            // Quantize frequency to 50Hz bands
            let quantized = UInt32(topPeak / 50.0)
            hashComponents.append(quantized)
        }
    }
    
    // Convert to hex string
    return hashComponents.map { String(format: "%02x", $0) }.joined()
}
```

**Example:**
- Top peaks from 10 sampled windows: [350, 400, 380, 420, 360, 390, 410, 370, 385, 395]
- Quantized (÷50): [7, 8, 7, 8, 7, 7, 8, 7, 7, 7]
- Hash: `"07080708070708070707"`

---

## The AudioFingerprint Data Structure

```swift
struct AudioFingerprint {
    let peaks: [[Float]]        // ~1290 windows × 5 peaks each
    let hash: String            // ~20 character hex string
    let analyzedDuration: Double // e.g., 30.0 seconds
}
```

---

## Fingerprint Comparison Algorithm

### Overview

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

### Sliding Window Offsets

**Why needed:** Same song may be trimmed differently:
- Song A starts with 3-second intro
- Song B has intro removed

```
Song A: [INTRO------|=====ACTUAL SONG======================]
Song B: [=====ACTUAL SONG======================]
         ↑
         Offset Song A by ~130 windows to align
```

**Parameters:**
| Parameter | Value | Explanation |
|-----------|-------|-------------|
| `maxOffset` | 400 windows | ~10 seconds at 44.1kHz |
| `stepSize` | 20 windows | ~0.5 seconds - balance speed/accuracy |

```swift
let maxOffset = min(400, min(peaks.count, other.peaks.count) / 3)
let stepSize = 20

for offset in stride(from: stepSize, to: maxOffset, by: stepSize) {
    let sim1 = compareAtOffset(selfOffset: offset, otherOffset: 0, other: other)
    let sim2 = compareAtOffset(selfOffset: 0, otherOffset: offset, other: other)
    bestSimilarity = max(bestSimilarity, sim1, sim2)
    
    if bestSimilarity >= 0.85 {
        return bestSimilarity  // Early exit
    }
}
```

### Window-by-Window Comparison

**Optimization:** Instead of comparing all ~1290 windows, sample every 10th window:

```swift
let sampleStep = 10  // Compare ~130 windows instead of 1290

for i in stride(from: 0, to: compareLength, by: sampleStep) {
    let selfPeaks = peaks[selfStart + i]      // 5 frequencies
    let otherPeaks = other.peaks[otherStart + i]  // 5 frequencies
    
    // Count matching peaks
    var windowMatches = 0
    for selfPeak in selfPeaks {
        for otherPeak in otherPeaks {
            if abs(selfPeak - otherPeak) <= 50.0 {  // ±50 Hz tolerance
                windowMatches += 1
                break
            }
        }
    }
    
    // Window matches if ≥50% of peaks match
    if Float(windowMatches) / Float(selfPeaks.count) >= 0.5 {
        matchingWindows += 1
    }
    sampledWindows += 1
}

return matchingWindows / sampledWindows  // Similarity score
```

**Frequency Tolerance (±50 Hz):**
- Accounts for minor encoding differences
- MP3 compression can shift frequencies slightly
- Different sample rates after resampling

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
│   │   Only compare within same bucket (fast!)               │    │
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

### Phase 1: Hash Pre-Filtering

```swift
// Group by hash first
let hashGroups = Dictionary(grouping: fingerprintedTracks) { track -> String in
    track.fingerprint?.hash ?? ""
}

// Only compare tracks with matching hashes
for (hash, samehashTracks) in hashGroups {
    if samehashTracks.count >= 2 {
        // Detailed comparison within group
        // O(N) within small group instead of O(N²) for all tracks
    }
}
```

### Phase 2: Cross-Hash Comparison

**Why needed:** Hash quantization (50 Hz bands) might put very similar fingerprints in different buckets.

```swift
// Example: Two songs with top peaks at 348 Hz vs 352 Hz
// Quantized: 348/50 = 6, 352/50 = 7
// Different hash buckets! But they're the same song.

for track1 in unprocessedTracks {
    for track2 in allTracks {
        let similarity = track1.fingerprint!.similarity(to: track2.fingerprint!)
        if similarity >= threshold {
            // Found a match across different hash buckets!
        }
    }
}
```

---

## Configurable Parameters

| Parameter | Location | Values | Impact |
|-----------|----------|--------|--------|
| Sample Duration | UI Picker | 10s, 30s, 60s, Full | Accuracy vs Speed |
| Similarity Threshold | UI Slider | 50% - 100% | False positives vs misses |
| FFT Size | Code | 2048 | Frequency resolution |
| Hop Size | Code | 1024 (50% overlap) | Time resolution |
| Peaks per Window | Code | 5 | Fingerprint precision |
| Frequency Range | Code | 300-3000 Hz | Focus area |
| Frequency Tolerance | Code | ±50 Hz | Match flexibility |
| Sliding Window Max | Code | 400 windows (~10s) | Offset tolerance |

---

## Performance Optimizations

### 1. Parallel Processing
```swift
await withTaskGroup(of: Void.self) { group in
    // Process multiple files concurrently
    for track in tracks {
        group.addTask {
            await fingerprintService.generateFingerprint(for: track.url)
        }
    }
}
```

### 2. Hash Pre-Filtering
- O(N) grouping first vs O(N²) full comparison
- Only detailed comparison within same hash bucket

### 3. Sampled Comparison
- Compare every 10th window (130 vs 1290 comparisons)
- 10x speedup with minimal accuracy loss

### 4. Early Exit
- Stop sliding window search on ≥85% match
- Return immediately on high-confidence matches

### 5. Apple Accelerate Framework
- Hardware-accelerated SIMD operations
- vDSP FFT is one of the fastest implementations available

---

## Why This Works

### Key Insight: Spectral Stability

The frequency content of a song remains **remarkably stable** across:
- Different encodings (MP3, AAC, FLAC)
- Different bitrates (128, 256, 320 kbps)
- Minor pitch/speed variations

The peak frequencies in the 300-3000 Hz range (vocals, lead instruments) are the "DNA" of the song.

### What Changes Between Encodings

| Aspect | Impact on Fingerprint |
|--------|----------------------|
| Lower bitrate | Slightly reduces high-frequency peaks |
| MP3 vs AAC | Minor phase differences (ignored by magnitude) |
| Sample rate | Handled by frequency-based analysis |
| Stereo width | Ignored by mono conversion |
| Volume normalization | Ignored (we only use peak *frequencies*, not magnitudes for matching) |

---

## Comparison: Our Algorithm vs Chromaprint/AcoustID

| Aspect | Our Custom FFT | Chromaprint |
|--------|---------------|-------------|
| **Purpose** | Local duplicate detection | Global song identification |
| **Dependency** | None (pure Swift) | Requires fpcalc binary |
| **Database** | Compares locally | Queries AcoustID servers |
| **Speed** | Very fast (local) | Slower (network) |
| **Accuracy** | High for duplicates | Identifies exact songs |
| **Use Case** | Find duplicate files | Fix incorrect metadata |

---

## Summary

The custom audio fingerprinting algorithm:

1. **Decodes** audio to format-independent PCM
2. **Analyzes** frequency content using FFT in sliding windows
3. **Extracts** top 5 peak frequencies per window (300-3000 Hz)
4. **Generates** hash for fast pre-filtering
5. **Compares** using sliding window alignment with ±50 Hz tolerance
6. **Optimizes** with hash bucketing, sampling, and early exit

This enables detection of the same song encoded at different bitrates, in different formats, or with minor trimming - all without any external dependencies.
