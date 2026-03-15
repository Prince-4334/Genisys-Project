# 🧬 Ry-Wave Filter — Quantum DNA Encoder
### Science Fest 2026 Hackathon Submission

> **Encoding DNA sequences into quantum circuits using Ry-rotation gates and a CRy wave filter — with noise simulation on a real IBM backend.**

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
   - [Section 1 — Sequence Utilities](#section-1--sequence-utilities)
   - [Section 2 — Ry Encoding](#section-2--ry-encoding)
   - [Section 3 — Wave Filter](#section-3--wave-filter)
   - [Section 4 — Circuit Builders](#section-4--circuit-builders)
   - [Section 5 — Decoder](#section-5--decoder)
   - [Section 6 — Simulation](#section-6--simulation)
   - [Section 7 — Metrics](#section-7--metrics)
   - [Section 8 — Diagram Generation](#section-8--diagram-generation)
3. [How It Works — End to End](#how-it-works--end-to-end)
4. [Key Design Decisions](#key-design-decisions)
5. [Performance Optimisations](#performance-optimisations)
6. [Outputs](#outputs)
7. [Installation](#installation)
8. [Usage](#usage)
9. [Results Summary](#results-summary)
10. [File Structure](#file-structure)

---

## Overview

This project encodes a DNA sequence (stored in `sequence.txt`) into quantum circuits using **Ry-rotation gates** to represent each nucleotide, then applies a **CRy wave filter** and its inverse to simulate quantum processing. The circuits are run on both an ideal statevector simulator and a **noisy FakeSherbrooke IBM backend**, measuring how faithfully the sequence survives decoherence.

| Property | Value |
|---|---|
| Qubits per window | 6 (3 bases × 2 qubits) |
| Encoding scheme | Ry-rotation (2-qubit binary per base) |
| Wave filter | CRy(φ) forward + CRy(−φ) inverse |
| Wave angle φ | π/4 (45°) |
| Noise backend | FakeSherbrooke (IBM) |
| Fidelity method | Diagonal trick — ρ[idx, idx] |

---

## Architecture

The codebase is divided into 8 clearly separated sections, each responsible for a distinct stage of the pipeline.

---

### Section 1 — Sequence Utilities

```python
def clean_dna(s)
def make_windows(seq, w)
def _trunc(s, n)
```

**Purpose:** Prepares the raw DNA string for quantum processing.

- `clean_dna` strips any non-ATGC characters and uppercases the input, ensuring only valid nucleotide symbols enter the pipeline.
- `make_windows` splits the cleaned sequence into fixed-width chunks of size `WINDOW_SIZE` (default: 3 bases). The last window is right-padded with `'A'` if the sequence length is not a multiple of 3. It also records `real_lens` — the true (unpadded) length of each window — so padding bases are never decoded as real sequence.
- `_trunc` is a display helper that caps long strings at `DISPLAY_COLS` (60 chars) for clean terminal output.

**Why sliding windows?** Encoding the entire sequence as one circuit would require `len(seq) × 2` qubits. The window approach caps qubit usage at a constant `N_QUBITS = 6`, achieving up to a `len(seq) × 2 / 6` qubit reduction.

---

### Section 2 — Ry Encoding

```python
BASE_MAP = {'A': 0, 'T': 1, 'G': 2, 'C': 3}

def ry_encode_base(qc, base, msb_qubit, lsb_qubit)
def ry_encode_window(qc, bases)
```

**Purpose:** Maps each nucleotide to a 2-qubit computational basis state using Ry(π) rotations.

Each base is assigned a 2-bit binary index:

| Base | Index | MSB qubit | LSB qubit |
|------|-------|-----------|-----------|
| A    | 00    | —         | —         |
| T    | 01    | —         | Ry(π)     |
| G    | 10    | Ry(π)     | —         |
| C    | 11    | Ry(π)     | Ry(π)     |

A Ry(π) rotation flips a qubit from |0⟩ to |1⟩. Bases encoded as `0` in a bit position get **no gate** — this is the "Ry pruning" optimisation: `A` requires zero gates, saving up to `2 × count(A)` gates across the whole sequence.

---

### Section 3 — Wave Filter

```python
def apply_wave_filter(qc, phi, n_bases)
def apply_inverse_wave_filter(qc, phi, n_bases)
```

**Purpose:** Applies an entangling CRy wave pattern across adjacent base-pairs, then reverses it — forming a unitary that is theoretically the identity.

- **Forward wave:** For each adjacent pair of bases `(i, i+1)`, applies `CRy(+φ)` controlled on the LSB qubit of base `i`, targeting the MSB qubit of base `i+1`. This propagates phase information across the strand.
- **Inverse wave:** Applies `CRy(−φ)` in reverse order, undoing the forward transformation.

**Net effect on ideal hardware:** Wave + Inverse = Identity, so the ideal decoded output perfectly matches the input. On **noisy hardware**, the round-trip reveals decoherence — the fidelity drop between ideal and noisy is a direct measure of gate noise.

---

### Section 4 — Circuit Builders

```python
def build_wave_window_circuit(bases, phi, measure)
def build_fidelity_circuit(bases, phi)
```

**Purpose:** Assembles complete Qiskit `QuantumCircuit` objects for a given window.

- `build_wave_window_circuit` constructs the full encode → wave → inverse → measure circuit. Barriers are inserted between stages for readability and to prevent the transpiler from merging gate layers across logical sections.
- `build_fidelity_circuit` builds the same circuit **without measurement**, used for density matrix fidelity computation.

Both functions accept `phi` as a parameter, enabling future experiments with different wave angles.

---

### Section 5 — Decoder

```python
def decode_bitstring(bitstring, n_bases)
def recover_sequence(all_counts, real_lens)
```

**Purpose:** Converts shot-measurement bitstrings back into DNA bases.

- `decode_bitstring` reverses the Qiskit bit-ordering convention (LSB = leftmost in string, so the string is reversed before indexing). For each base position, it reads the 2-bit pair `(MSB, LSB)` and looks up the `IDX_TO_BASE` map.
- `recover_sequence` iterates over all windows, selects the **most frequent bitstring** (majority vote over `SHOTS=2048` shots) as the decoded result, and concatenates windows to reconstruct the full strand.

The majority-vote strategy adds implicit error correction — transient noise affecting a minority of shots does not corrupt the decoded base.

---

### Section 6 — Simulation

```python
def simulate_windows(windows, real_lens, noisy, shots, noise_model)
def compute_fidelity_fast(window, sim_noisy, noise_model)
def compute_all_fidelities(windows, real_lens, backend)
def _target_index(bases)
```

**Purpose:** Runs all circuits on ideal and noisy simulators, and computes per-window quantum state fidelity.

#### `simulate_windows`
Runs every window circuit using either:
- `AerSimulator(method='statevector')` — ideal, noiseless
- `AerSimulator(method='density_matrix', noise_model=...)` — noisy, using `NoiseModel.from_backend(FakeSherbrooke)`

Circuits are transpiled to native gate sets (`x, rz, sx, ry, cx, reset, measure, id`) at `optimization_level=3` before execution.

#### `_target_index`
Computes the expected computational basis state index after encode + wave + inverse. Since the wave and inverse cancel perfectly on ideal hardware, the output state is exactly the same basis state as the input encoding — making this analytically predictable without simulation.

#### `compute_fidelity_fast` — The Diagonal Trick
Instead of running two simulations (ideal + noisy) and calling `state_fidelity(ideal, noisy)`, this function exploits the fact that the ideal output is a **pure computational basis state** |ψ⟩. For such states:

```
F(|ψ⟩, ρ_noisy) = ⟨ψ|ρ_noisy|ψ⟩ = ρ_noisy[idx, idx]
```

Only the diagonal element of the noisy density matrix is needed — cutting per-window simulation time roughly in half.

#### `compute_all_fidelities`
Deduplicates windows (many DNA sequences have repeated triplets), builds the noise model and simulator **once**, and reuses them. A progress bar tracks completion.

---

### Section 7 — Metrics

```python
def count_gates(qc)
def sequence_accuracy(orig, rec)
```

**Purpose:** Lightweight measurement utilities.

- `count_gates` iterates over `qc.data` and tallies each gate type — used to report Ry, CRy, CX, and depth statistics.
- `sequence_accuracy` computes base-level accuracy between original and recovered sequences (number of matching positions / total length).

---

### Section 8 — Diagram Generation

```python
def generate_all_diagrams(...)
```

**Purpose:** Produces four publication-quality visualisation PNGs on a dark `#0d1117` background.

| File | Content |
|---|---|
| `ry_wave_circuit_logical.png` | Hand-drawn logical circuit showing encode, wave, inverse, and measurement stages for a representative window |
| `ry_wave_circuit_transpiled.png` | Gate-by-gate view of the transpiled circuit, coloured by native gate type |
| `ry_wave_circuit_sliding.png` | Three consecutive windows shown on shared qubit wires, illustrating the sliding-window approach |
| `ry_wave_dashboard.png` | Full analysis dashboard: per-window fidelity bar chart, fidelity gauge, gate summary table, Ry-gate usage by base, fidelity histogram, nucleotide composition pie, and decoded strand comparison |

Colour scheme uses base-specific colours (A=green, T=orange, G=blue, C=red) throughout all diagrams for visual consistency.

---

## How It Works — End to End

```
sequence.txt
     │
     ▼
clean_dna()          ← remove non-ATGC characters
     │
     ▼
make_windows()       ← split into 3-base windows, pad last window
     │
     ├──────────────────────────────────────────────────┐
     ▼                                                  ▼
build_wave_window_circuit()                  build_fidelity_circuit()
   Ry encode → CRy wave → CRy⁻¹ → Measure      (no measurement)
     │                                                  │
     ├── AerSimulator (statevector)          AerSimulator (density_matrix)
     │         ideal counts                         noisy ρ
     │                                                  │
     ▼                                                  ▼
recover_sequence()                         ρ[idx, idx]  → fidelity per window
   majority vote per window
     │
     ▼
sequence_accuracy()  ← compare original vs recovered
     │
     ▼
generate_all_diagrams()  ← 4 PNG outputs
```

---

## Key Design Decisions

**2 qubits per base** — The 4 DNA bases map cleanly to 2-bit binary (00–11), making the encoding lossless and reversible.

**Wave + inverse = identity** — This deliberate round-trip design means 100% accuracy on ideal hardware is guaranteed by construction. Any error in the noisy simulation is purely attributable to gate noise, not encoding loss.

**Window-based processing** — Keeps the qubit count constant at 6 regardless of sequence length, making the approach scalable to genome-scale sequences.

**Majority-vote decoding** — Running 2048 shots and taking the most frequent bitstring provides implicit error correction at the measurement level.

**Diagonal fidelity trick** — Eliminates one full density matrix simulation per window, cutting fidelity computation time by ~50%.

**Deduplication** — Sequences with repeated triplets (common in biology) only simulate each unique window once, with results cached and reused.

---

## Performance Optimisations

| Fix | Description | Impact |
|-----|-------------|--------|
| FIX 1 | `NoiseModel` built once, shared across all calls | Eliminates repeated backend parsing |
| FIX 2 | `AerSimulator` instances built once and reused | Removes constructor overhead per window |
| FIX 3 | Diagonal fidelity trick: `F = ρ[idx,idx]` | ~50% fewer simulations for fidelity |
| FIX 4 | Progress bar during fidelity computation | Prevents the run appearing hung |
| FIX 5 | Output truncated to `DISPLAY_COLS=60` chars | Keeps terminal output readable for long sequences |

---

## Outputs

Running `main()` produces:

**Terminal output:**
- Sequence statistics (length, GC content, qubit reduction)
- Gate anatomy for a representative window
- Ry gate pruning stats
- Per-window fidelity (mean, min)
- Ideal and noisy recovery accuracy
- Error map showing mismatched positions
- Decoded strand alignment
- Scalability projection table

**PNG files:**
- `ry_wave_circuit_logical.png`
- `ry_wave_circuit_transpiled.png`
- `ry_wave_circuit_sliding.png`
- `ry_wave_dashboard.png`

---

## Installation

```bash
# Python 3.9+
pip install qiskit qiskit-aer qiskit-ibm-runtime numpy matplotlib
```

> Note: `FakeSherbrooke` is imported from `qiskit_ibm_runtime.fake_provider` if available, falling back to `qiskit.providers.fake_provider`.

---

## Usage

1. Place your DNA sequence in `sequence.txt` (plain text, any case, non-ATGC characters are ignored).
2. Run:

```bash
python solution.py
```

To change window size, wave angle, or shot count, edit the constants at the top of the file:

```python
WINDOW_SIZE = 3       # bases per window
WAVE_PHI    = np.pi/4 # CRy entanglement angle
SHOTS       = 2048    # measurement shots per window
```

---

## Results Summary

| Metric | Value |
|---|---|
| Qubits used | 6 (constant, regardless of sequence length) |
| Ideal recovery accuracy | 100% |
| Mean state fidelity (noisy) | ~0.95+ (depends on sequence) |
| Fidelity method | Diagonal trick — single density matrix sim per window |
| Noise backend | FakeSherbrooke (IBM calibration data) |

---

## File Structure

```
.
├── solution.py       # Main pipeline (all 8 sections)
├── sequence.txt      # Input DNA sequence
├── README.md         # This file
└── outputs/
    ├── ry_wave_circuit_logical.png
    ├── ry_wave_circuit_transpiled.png
    ├── ry_wave_circuit_sliding.png
    └── ry_wave_dashboard.png
```
