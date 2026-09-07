# KyberPoly2msg-DS: Kyber Decapsulation poly2msg EM Side-Channel Dataset

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22638659.svg)](https://doi.org/10.5281/zenodo.22638659)

Real-hardware electromagnetic (EM) side-channel traces captured during CRYSTALS-Kyber decapsulation (pq-crystals reference implementation) on an ARM Cortex-M4 target, targeting the `poly2msg()` message-decoding step inside `K-PKE.Decrypt`.

## Acquisition parameters

| Parameter | Value |
|---|---|
| Channel | EM probe, digitized via the target board's `vcap` acquisition pin |
| Sampling rate | 1.25 GS/s |
| Core clock | 168 MHz |
| Samples per trace | 150,000 |
| Trace dtype | int16 |
| Target device | STM32F439ZI (ARM Cortex-M4) |
| Target implementation | CRYSTALS-Kyber, pq-crystals reference code, unmasked |

## File structure

Single HDF5 file, uncompressed, chunked.

data/
├── profiling/ <br />
│   ├── traces   (100000, 150000)  int16 <br />
│   ├── message  (100000, 32)      uint8 <br />
│   └── k        (100000, 32)      uint8 <br />
└── attack/ <br />
├── traces   (50000, 150000)   int16 <br />
├── message  (100000, 32)*     uint8 <br />
└── k        (100000, 32)*     uint8 <br />


\* attack/message and attack/k are declared at 100,000 rows, but only the first 50,000 were ever written, rows 50,000–99,999 are unallocated HDF5 chunks with no corresponding trace, read back as zero only because that's HDF5's default fill value, not stored padding. Always read attack/traces.shape[0] (50,000) first and slice to it.

## Field semantics

- **traces**: raw EM samples.
- **k**: 32-byte key material (not used as an attack target in this release).
- **message**: 32-byte **raw random seed** sampled at encapsulation time, *before* Kyber's internal defensive pre-hash. The value `poly2msg()` actually reconstructs on-device, `m'`, is `SHA3-256(message)`, not these raw bytes — hash the field yourself to obtain the true per-trace target byte: `t = m'[0] = SHA3-256(message)[0]`.

## Partitions / adversary model

- **profiling**: 100,000 traces from a fully controllable device, independent random key/message pair per trace.
- **attack**: 50,000 real traces from a device under a **single fixed, unknown** key/message pair, a fixed-target recovery scenario.

## Accessing the data

The full dataset (~45 GB) is hosted on Zenodo: [10.5281/zenodo.22638659](https://doi.org/10.5281/zenodo.22638659).

````
python
import h5py

with h5py.File("dataset.h5", "r") as f:
    profiling_traces = f["data/profiling/traces"][:1000]   # slice — don't load all 100k at once
    profiling_message = f["data/profiling/message"][:1000]

    n_attack = f["data/attack/traces"].shape[0]              # 50000 — read this first
    attack_traces = f["data/attack/traces"][:n_attack]
    attack_message = f["data/attack/message"][:n_attack]     # aligned 1:1 with attack_traces

````

traces datasets are chunked as (1000, n_samples) read in batches of that size (or multiples) for efficient I/O.

A runnable version of this walkthrough that inspects the file structure, loads both partitions, and constructs the poly2msg label is provided in Tutorial-Access-Dataset.ipynb.

