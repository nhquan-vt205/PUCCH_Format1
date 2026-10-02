# PUCCH Format 1 Receiver — MLE and Data-Path Specification

Version: 1.0  
Status: Datapath contract freeze

## 1. Purpose

This document freezes how ideal candidate sequences and extracted noisy samples are represented, aligned, compared, and reduced to the UCI decision.

## 2. Core data model

The detector uses two vectors of equal length:

```text
Z_std[k]  = { z_std[k][0], z_std[k][1], ... z_std[k][N-1] }
Z_noisy   = { z_noisy[0],   z_noisy[1],   ... z_noisy[N-1] }
```

where:

$$
N=N_{RE,data}
$$

For the baseline `N_symb=14`, no hopping:

$$
N_{SF}=7, \qquad N=84
$$

For other lengths/hopping modes, `N` follows the PUCCH data-symbol rules.

## 3. Candidate count

```text
uci_bit_count = 1 -> K=2
uci_bit_count = 2 -> K=4
```

No other candidate count is valid for the current scope.

## 4. Sample alignment contract

For every valid `q`:

```text
z_std[k][q]
       |
       | same physical RE
       v
z_noisy[q]
```

The Standard-Z Generator and RE Extractor MUST agree on:

- hop index;
- PUCCH data symbol index;
- OFDM symbol position;
- PRB index;
- subcarrier index;
- sample order.

A mismatch in sample alignment is a functional error, even if both individual generators are locally correct.

## 5. Canonical sample descriptor

Each sample `q` is conceptually described by:

```text
q
m'              hop index
m               PUCCH data spreading-symbol index
l               relative PUCCH OFDM symbol index
n               subcarrier 0..11
prb             selected PRB
```

The implementation MAY avoid carrying all fields physically, but the mapping MUST be deterministic.

## 6. Standard candidate sample

For candidate `k` at sample `(m',m,n)`:

$$
z_{std,k}(m',m,n)=w_i(m)\,d_k\,r_{u,v}^{(\alpha_l,\delta)}(n)
$$

This is the canonical ideal complex sample.

The sequence generation MUST respect per-symbol `alpha_l` and per-hop sequence parameters where applicable.

## 7. Noisy extracted sample

The RE Extractor obtains:

$$
z_{noisy}(m',m,n)=\hat Y(m',m,n)
$$

where `Y_hat` is the already estimation/equalization-processed resource-grid sample.

No additional sequence removal occurs before MLE.

## 8. Fixed-point candidate representation

The functional algorithm is complex floating point; RTL uses fixed point.

The baseline I/Q input format is signed Q1.15, 16 bit.

The Standard-Z Generator SHOULD preserve enough fractional precision that quantization error does not materially change the candidate ordering in noise-free tests.

An implementation may use one of:

- direct LUT-based I/Q generation;
- phase ROM + CORDIC;
- phase accumulator + CORDIC;
- symmetry/sign/swap optimizations for the BPSK/QPSK symbol.

The implementation MUST NOT change the functional sample values beyond the documented fixed-point tolerance.

## 9. MLE metric

Normative metric:

$$
E_k=\sum_q \left|z_{noisy}[q]-z_{std,k}[q]\right|^2
$$

I/Q implementation:

$$
D_{I,k,q}=z_{noisy,I}[q]-z_{std,k,I}[q]
$$

$$
D_{Q,k,q}=z_{noisy,Q}[q]-z_{std,k,Q}[q]
$$

$$
E_k=\sum_q(D_{I,k,q}^2+D_{Q,k,q}^2)
$$

Decision:

$$
q_{best}=\arg\min_k E_k
$$

The selected `k_best` MUST map to the candidate bit vector defined in the algorithm specification.

## 10. Streaming implementation rule

MLE does not need to store the entire vectors if the architecture can present matching samples at the same `q`.

A resource-efficient implementation MAY compute:

```text
for q = 0 .. N-1:
    read z_noisy[q]
    for each k:
        read/generate z_std[k][q]
        accumulate squared distance into metric[k]
```

At `q=N-1`, the metric accumulators become final candidate errors and are passed to a comparator tree.

## 11. Recommended implementation structure

```text
                 +--------------------+
                 | sample_q generator |
                 +---------+----------+
                           |
              +------------+------------+
              |                         |
              v                         v
     z_std candidate generator    z_noisy stream
              |                         |
              +------------+------------+
                           v
                  +------------------+
                  | squared-distance |
                  | metric engines   |
                  +--------+---------+
                           |
                     metric[0..K-1]
                           |
                           v
                  +------------------+
                  | min comparator    |
                  +--------+---------+
                           |
                        k_best
                           |
                           v
                       UCI bits
```

## 12. Candidate parallelism

For `K=4`, the simplest baseline is four metric accumulators operating in parallel.

For lower area, the implementation MAY time-multiplex a smaller number of metric engines across candidates.

The architecture contract is independent of that choice.

## 13. Metric width derivation

Metric width MUST be derived before RTL freeze.

For each difference:

- determine the signed difference width after alignment/extension;
- square it;
- add I and Q terms;
- accumulate over maximum `N_RE,data`;
- add sufficient headroom for saturation/rounding policy.

The previous 26-bit accumulator width in the old coherent-combining architecture is not a valid default for this metric accumulator.

## 14. Overflow and saturation

The metric accumulator MUST use one explicitly documented policy:

- widened no-overflow accumulator; or
- saturating accumulator.

Wrap-around is forbidden because it can invert candidate ordering.

If saturation occurs, `error` SHOULD be asserted and `result_valid` SHOULD be suppressed for that transaction.

## 15. Metric ordering and second-best candidate

The comparator MUST produce:

```text
best candidate
second-best candidate
metric_best
metric_second
```

where lower Euclidean error is better.

If software-facing status historically expects a "higher-is-better" confidence metric, that derived quantity MUST be defined separately; the canonical MLE metric remains `E_k` with lower-is-better semantics.

## 16. Tie handling

Tie handling MUST be deterministic.

Recommended rule:

```text
if E_a == E_b, choose smaller candidate index
```

This rule applies to both best and second-best ordering.

## 17. Noise-free invariants

If the grid contains an ideal candidate sequence exactly:

$$
z_{noisy}[q]=z_{std,k_0}[q]
$$

then:

$$
E_{k_0}=0
$$

within quantization tolerance and:

$$
k_{best}=k_0
$$

for every valid configuration.

This is the primary end-to-end detector invariant.

## 18. What the architecture intentionally does not use

The following are NOT inputs to MLE in this architecture:

- scalar coherent `Z`;
- sign of `Re(Z)`;
- sign of `Im(Z)`;
- CORDIC-derotated received samples;
- OCC-despread received samples.

Those belong to the superseded matched-filter implementation path.
