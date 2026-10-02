# PUCCH Format 1 Receiver — System Architecture Specification

Version: 1.0  
Status: Design-ready architecture freeze

## 1. Purpose

This document freezes the top-level architecture for a PUCCH Format 1 gNB receiver using direct candidate-sequence comparison.

The architecture starts from a resource grid that has already undergone channel estimation and equalization and ends at UCI bits.

The core design principle is:

> Generate all ideal PUCCH candidate sequences independently from configuration, extract the corresponding equalized RE samples from the resource grid, and compare the two sequences directly in the MLE block.

The architecture MUST NOT reinterpret the detector as a matched-filter path that first converts the received sequence into one scalar coherent statistic.

## 2. Scope and baseline

The specification follows the current project baseline:

- PUCCH Format 1.
- FR1.
- 30 kHz SCS.
- Normal CP.
- One PRB for the PUCCH resource.
- Up to 2 UCI bits.
- 1 Rx antenna baseline.
- One active UE context baseline.
- Channel estimation and equalization are outside the detector core.
- Equalized grid samples are complex signed fixed-point values using the project baseline convention of Q1.15, 16-bit I/Q.

Configurable PUCCH parameters MUST remain configurable even when the first RTL revision verifies only a baseline operating point.

## 3. Top-level architecture

```text
                                  CONTROL PLANE
                         +---------------------------+
                         |        AXI4-Lite          |
                         +-------------+-------------+
                                       |
                                       v
                              +----------------+
                              |   Config RAM    |
                              | active/shadow   |
                              +--------+--------+
                                       |
                                       v
                              +----------------+
                              |  Config Decode |
                              +---+---------+--+
                                  |         |
                         config   |         | config/timing
                                  |         |
                                  v         v
                       +----------------+  +----------------+
                       | Standard-Z Gen |  |  RG RAM / Grid |
                       |                |  |    Buffer      |
                       +-------+--------+  +-------+--------+
                               |                   |
                        Z_std[k][q]          AXI4-Full data
                               |                   |
                               |                   v
                               |            +--------------+
                               |            | RE Extractor |
                               |            +------+-------+
                               |                   |
                               |             z_noisy[q]
                               |                   |
                               +---------+---------+
                                         v
                                  +-------------+
                                  |     MLE     |
                                  +------+------+ 
                                         |
                                     UCI bits

                                  STATUS / RESULT
                                    -> AXI4-Lite
```

## 4. Block responsibilities

### 4.1 Config RAM

Config RAM is a small register-file-like memory accessed through AXI4-Lite.

It MUST:

- store all PUCCH configuration required by the detector;
- provide an active configuration snapshot to compute blocks;
- support software-visible status/result registers;
- prevent an in-flight PUCCH operation from observing partially updated configuration.

Recommended implementation: shadow configuration registers are writable while idle; a `START` operation snapshots them into an active bank.

### 4.2 Config Decode

Config Decode derives implementation-ready parameters from active configuration and current timing.

It MUST provide, directly or through submodule interfaces:

- modulation mode (BPSK/QPSK);
- candidate count (`K=2` or `K=4`);
- `u`, `v`;
- per-data-symbol `alpha` / phase parameters;
- `N_SF` and `m'` handling;
- OCC index and OCC sequence values;
- PUCCH RE coordinates;
- canonical sample ordering metadata.

Config Decode does not access RG RAM and does not perform MLE.

### 4.3 Standard-Z Generator

This is the reference/candidate sequence generation block.

For each valid candidate `k`, it MUST generate the ideal sequence after block-wise spreading:

```text
candidate d_k
    -> low-PAPR reference sequence
    -> multiply by d_k
    -> block-wise spreading by w_i(m)
    -> z_std[k][q]
```

The output sequence MUST correspond exactly to the PUCCH data RE order used by the RE Extractor.

The Standard-Z Generator MUST NOT consume noisy RG samples.

The existing phase-accumulator concept belongs here: phase-domain sequence generation MAY be implemented with a base-phase LUT plus cyclic-shift phase accumulation. A CORDIC MAY be used to turn generated phase into I/Q. If implemented, it is a realization detail of Standard-Z generation, not a receive-side derotation stage.

### 4.4 RG RAM / Grid Buffer

RG RAM stores the resource grid after channel estimation and equalization.

For the baseline, each complex sample is:

```text
{ I[15:0], Q[15:0] }
```

with signed Q1.15 I/Q.

The grid memory is accessed by the data plane using AXI4-Full or an equivalent internal memory subsystem.

### 4.5 RE Extractor

RE Extractor maps PUCCH configuration and timing into resource-grid addresses, reads only the UCI data REs, skips DM-RS/empty REs, and outputs the selected equalized samples in canonical order.

Its output is the noisy observed sequence:

```text
z_noisy[q]
```

No reference derotation, OCC despreading, or coherent accumulation is performed in this block.

### 4.6 MLE

MLE receives:

- `Z_std[k]` for all valid candidates;
- `Z_noisy`;
- candidate count / UCI bit count.

It computes a metric for every candidate and selects the candidate with the minimum canonical error.

The output candidate index is converted to UCI bits using the frozen mapping in the algorithm specification.

## 5. End-to-end dataflow

### Standard branch

```text
active config + timing
       |
       v
sequence parameters
       |
       v
candidate d_k
       |
       v
low-PAPR sequence r_uv^(alpha,delta)
       |
       v
block-wise spreading w_i(m)
       |
       v
Z_std[k][q]
```

### Noisy branch

```text
Equalized resource grid in RG RAM
       |
       v
PUCCH address generation
       |
       v
RE extraction
       |
       v
z_noisy[q]
       |
       v
Z_noisy vector
```

### Decision

```text
Z_std[0..K-1] + Z_noisy
                 |
                 v
                MLE
                 |
                 v
              k_best
                 |
                 v
             UCI bits
```

## 6. Explicit non-goals

The following functions are NOT part of the noisy-data path in this architecture:

- reference derotation of `z_noisy`;
- OCC despreading of `z_noisy`;
- coherent accumulation of `z_noisy` into a scalar `Z`;
- candidate decision from `sign(Re(Z))` or `sign(Im(Z))`.

Those operations belong to the previous matched-filter architecture and MUST NOT be reintroduced into the new top-level datapath unless the architecture is intentionally revised.

## 7. Timing ownership

The architecture uses system timing inputs:

```systemverilog
frame_idx
slot_idx
symbol_idx
```

PUCCH configuration determines the PUCCH start symbol, length, PRB and hopping behavior.

The Standard-Z Generator and RE Extractor MUST use the same configuration snapshot and the same timing interpretation so that candidate sequence sample `q` corresponds to the same physical RE as noisy sample `q`.

## 8. Configuration snapshot rule

When `START` is accepted:

1. shadow configuration is validated;
2. active configuration is latched;
3. candidate generation and extraction use only the active snapshot;
4. writes to shadow configuration during `busy=1` do not affect the current operation;
5. completion produces result/status;
6. the next operation may use the updated configuration.

This rule removes reconfiguration ambiguity between the AXI4-Lite control plane and the compute datapath.

## 9. Required top-level blocks

The RTL hierarchy SHOULD contain at least:

```text
pucch_f1_receiver_top
├── axi4lite_cfg_if
├── config_ram
├── config_decode
├── standard_z_generator
├── axi4full_grid_if
├── rg_ram
├── re_extractor
├── mle_detector
└── control_fsm
```

Sub-block partitioning inside `standard_z_generator`, `config_decode`, and `re_extractor` is implementation-defined, provided the external block contracts remain unchanged.
