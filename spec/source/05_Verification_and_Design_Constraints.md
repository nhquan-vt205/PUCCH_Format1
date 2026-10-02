# PUCCH Format 1 Receiver — Verification and Design Constraints

Version: 1.0  
Status: Design/verification freeze

## 1. Verification objective

Verification MUST prove both:

1. the Standard-Z Generator creates the correct ideal candidate sequence;
2. the RE Extractor presents the same physical REs in the same order;
3. the MLE selects the correct candidate under noise and quantization.

## 2. Golden model architecture

The golden model SHOULD mirror the RTL functional blocks:

```text
config
  -> parameter derivation
  -> candidate modulation
  -> low-PAPR sequence
  -> cyclic shift
  -> block-wise spreading
  -> Z_std[k]

resource grid
  -> address/reference mapping
  -> RE extraction
  -> Z_noisy

Z_std[k] + Z_noisy
  -> Euclidean metrics
  -> candidate decision
  -> UCI bits
```

The golden model MUST NOT hide the new architecture behind the old scalar-Z matched-filter model.

## 3. Required golden-model outputs

For every test transaction, the golden model SHOULD be able to dump:

```text
active_config
u, v
n_cs / alpha per relevant symbol
candidate d_k
phi_u(n)
r_uv^(alpha,delta)(n)
w_i(m)
z_std[k][q]
RE physical coordinates[q]
z_noisy[q]
metric[k]
k_best
uci_bits
```

## 4. Test classes

### 4.1 Candidate-generation tests

Exhaustively verify:

- 1-bit BPSK candidate mapping;
- 2-bit QPSK candidate mapping;
- low-PAPR `phi_u(n)` table;
- cyclic shift calculation;
- group hopping modes;
- sequence number `v` behavior;
- OCC tables and `N_SF` dependence;
- candidate sample order.

### 4.2 RE-extraction tests

Verify:

- start symbol;
- PUCCH length 4..14;
- first-hop PRB;
- second-hop PRB;
- no hopping;
- intra-slot hopping;
- DM-RS exclusion;
- first/last sample signaling;
- sample index alignment.

### 4.3 End-to-end noise-free tests

Generate each candidate's ideal `z_std[k]` and write it into the corresponding resource-grid data REs.

Then verify:

```text
Z_noisy == Z_std[k_true]
metric[k_true] ~= 0
k_best == k_true
UCI_out == UCI_in
```

All supported candidate values MUST be tested.

### 4.4 AWGN tests

Perform SNR sweeps over a meaningful range, including difficult low-SNR points.

Record:

- candidate error metrics;
- best/second-best separation;
- bit decisions;
- numerical saturation events.

### 4.5 Equalization-error tests

Inject:

- amplitude error;
- common phase error;
- per-RE phase error;
- residual channel-estimation error.

These tests establish the limits of the direct sequence-comparison architecture and verify the input contract.

## 5. Sample-by-sample comparison requirements

The verification environment MUST compare:

```text
reference candidate sequence
vs
RTL Standard-Z Generator output
```

sample by sample in the same canonical order.

It MUST also compare:

```text
expected physical RE coordinate[q]
vs
RTL RE extractor address[q]
```

This is essential because a sample-order error can produce a plausible-looking waveform while making the MLE fail.

## 6. Assertions

Useful RTL assertions include:

### Configuration

```text
start only when !busy
active config stable while busy
uci_bit_count in {1,2}
occ_index < N_SF
start_symbol + nsymb <= 14 for baseline
```

### Stream alignment

```text
sample_q increments only on valid data RE
sample_first implies sample_q == 0
sample_last implies sample_q == N_RE,data-1
z_std and z_noisy use identical sample_q
```

### Result

```text
result_valid -> !busy
result_valid -> !error
k_best in valid candidate range
```

## 7. Architecture-level acceptance criteria

The design is accepted only when:

1. `Z_std[k]` matches the floating-point/reference model within agreed fixed-point tolerance.
2. RE addresses match the golden physical mapping.
3. `Z_noisy` sample order matches `Z_std` sample order.
4. Noise-free candidate injection always returns the intended UCI bits.
5. No candidate metric wraps around.
6. Tie handling is deterministic.
7. AXI4-Lite configuration does not change an active transaction.
8. AXI4-Full grid writes are visible to the detector according to the defined memory coherency rule.
9. `result_valid` is asserted only for a completed valid transaction.

## 8. Design constraints

### Must freeze before RTL

- exact AXI bus widths used by implementation;
- register offsets;
- RG RAM address mapping;
- active/shadow configuration behavior;
- Standard-Z sample fixed-point format;
- MLE metric width;
- MLE metric saturation/rounding;
- pipeline latency;
- candidate scheduling (parallel vs time-multiplexed);
- memory coherency/visibility boundary between AXI4-Full writer and detector reader.

### Must not be inferred by Claude

Claude MUST NOT invent:

- a scalar coherent `Z` datapath;
- receive-side derotation;
- receive-side OCC despreading;
- a 26-bit accumulator copied from the previous design;
- a sign-only BPSK/QPSK detector based on scalar `Z`;
- a replacement for AXI4-Full with AXI4-Lite single-sample accesses;
- a different candidate ordering than the frozen UCI mapping.

## 9. Resolved ambiguities from the previous documents

### A. Meaning of `Z`

Resolved: `Z_std[k]` and `Z_noisy` are sequence/vector objects. The old scalar coherent statistic `Z` is not part of the new architecture.

### B. Location of phase accumulator

Resolved: if used, the phase accumulator belongs inside Standard-Z generation to construct the ideal reference phase. It is not a receive-side derotation engine.

### C. Location of OCC

Resolved: OCC/block-wise spreading is applied in Standard-Z generation because `Z_std` is defined after block-wise spreading. RE extraction does not apply OCC to the noisy samples.

### D. Meaning of RE extraction

Resolved: RE extraction is a selection/addressing operation that converts the equalized resource grid into the ordered noisy PUCCH data sequence.

### E. Need for coherent accumulation

Resolved: no scalar coherent accumulation is required for the direct-sequence MLE architecture.

### F. MLE input

Resolved: MLE compares `Z_noisy[q]` against all `Z_std[k][q]` over the complete data-RE sequence.

### G. MLE canonical metric

Resolved: Euclidean sum-of-squared complex error is the normative metric. Correlation is an implementation optimization only if equivalence is demonstrated.

### H. Memory roles

Resolved:

- Config RAM: control/configuration storage.
- RG RAM: equalized resource-grid storage.
- Z_std storage: optional implementation buffer inside Standard-Z/MLE path.
- Z_noisy storage: optional implementation buffer inside RE/MLE path.

## 10. Reference testbench transaction

A recommended transaction is:

```text
1. software writes configuration through AXI4-Lite;
2. software/common data-plane producer writes equalized RG data through AXI4-Full;
3. software asserts START;
4. Config RAM snapshots active configuration;
5. Standard-Z Generator emits all K candidates in canonical q order;
6. RE Extractor emits z_noisy[q] in the same q order;
7. MLE accumulates E_k;
8. comparator selects k_best;
9. candidate index maps to UCI bits;
10. result_valid asserts for one cycle;
11. status/metrics remain readable until cleared or next transaction.
```

## 11. Out-of-scope behavior

Unless explicitly re-added by a later architecture revision:

- multi-Rx combining;
- multiple simultaneous UE contexts;
- inter-slot spanning PUCCH occasions;
- interlace-based mapping;
- soft-output UCI;
- downstream channel decoding beyond the PUCCH F1 symbol decision.

## 12. Final design rule

The RTL implementation MUST be traceable to this chain:

```text
CONFIG
  -> candidate generation
  -> Standard-Z sequence

EQUALIZED RESOURCE GRID
  -> RE extraction
  -> Noisy sequence

Standard-Z sequence + Noisy sequence
  -> MLE
  -> UCI
```

Any design proposal that introduces additional processing between RE extraction and MLE MUST be treated as an architectural change and documented explicitly before implementation.
