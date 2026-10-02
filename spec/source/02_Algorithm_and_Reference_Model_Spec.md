# PUCCH Format 1 Receiver — Algorithm and Reference Model Specification

Version: 1.0  
Status: Functional algorithm freeze

## 1. Functional objective

Recover the UCI bits by comparing the equalized PUCCH data RE sequence against all valid ideal candidate sequences.

The algorithm operates on a per-data-RE sequence basis. It does not reduce the received sequence to one scalar matched-filter statistic before candidate comparison.

## 2. Notation freeze

To avoid collision with the legacy use of `z` and scalar `Z`, this document uses:

- `d_k`: modulation symbol for candidate `k`.
- `z_std[k][q]`: ideal candidate sequence sample at canonical sample `q`.
- `z_noisy[q]`: equalized grid sample extracted at canonical sample `q`.
- `Z_std[k]`: vector containing all `z_std[k][q]` samples.
- `Z_noisy`: vector containing all `z_noisy[q]` samples.
- `E_k`: Euclidean MLE error metric.
- `k_best`: selected candidate index.

`Z_std[k]` and `Z_noisy` are vectors/sequences, not scalar coherent statistics.

## 3. PUCCH Format 1 candidate modulation

### 3.1 One-bit UCI

Candidate set:

```text
k=0 -> b(0)=0 -> d_0 = +(1+j)/sqrt(2)
k=1 -> b(0)=1 -> d_1 = -(1+j)/sqrt(2)
```

### 3.2 Two-bit UCI

Candidate set:

```text
k=0 -> 00 -> d_0 = +(1+j)/sqrt(2)
k=1 -> 01 -> d_1 = +(1-j)/sqrt(2)
k=2 -> 10 -> d_2 = -(1-j)/sqrt(2) = (-1+j)/sqrt(2)
k=3 -> 11 -> d_3 = -(1+j)/sqrt(2)
```

Bit ordering is frozen as:

```text
uci_bits[0] = b(0)
uci_bits[1] = b(1)   // valid only for 2-bit UCI
```

## 4. Sequence parameter derivation

The receiver MUST derive sequence parameters using the configuration/timing rules already established in the project specification:

```text
n_ID / hopping_id
      |
      +--> group hopping mode -> f_gh
      |
      +--> sequence shift      -> f_ss
      |
      +--> sequence group      -> u
      |
      +--> sequence number     -> v
      |
      +--> cyclic-shift term   -> n_cs
      |
      +--> m0 + n_cs -> alpha per PUCCH OFDM symbol
```

For the current baseline, the existing project rules remain authoritative for:

- `u`, `v` generation;
- Gold sequence initialization;
- `phi_u(n)` table for the length-12 low-PAPR sequence;
- `alpha` calculation;
- `delta = 0` baseline;
- PUCCH data/DM-RS symbol pattern;
- `N_SF` lookup versus PUCCH length and hopping.

## 5. Low-PAPR sequence

For subcarrier `n=0..11`:

$$
r_{u,v}(n)=e^{j\frac{\pi}{4}\phi_u(n)}
$$

With cyclic shift:

$$
r_{u,v}^{(\alpha,\delta)}(n)=e^{j\left(\frac{\pi}{4}\phi_u(n)+\alpha(n+\delta)\right)}
$$

For each PUCCH data OFDM symbol, the applicable `alpha` is derived from its symbol index according to the frozen project rules.

## 6. Block-wise spreading and standard candidate sequence

For candidate `k`:

$$
y_{\mathrm{std},k}(n)=d_k\,r_{u,v}^{(\alpha,\delta)}(n)
$$

Then block-wise spreading is:

$$
z_{\mathrm{std},k}(m',m,n)=w_i(m)\,y_{\mathrm{std},k}(n)
$$

This equation defines the sequence that the Standard-Z Generator MUST output.

The standard candidate sequence therefore contains all effects that exist before mapping to the physical resource grid:

```text
modulation d_k
   x
low-PAPR reference r_uv^(alpha,delta)
   x
block-wise OCC w_i(m)
   = z_std[k][m',m,n]
```

No noise, channel, or equalizer effect is included in `z_std`.

## 7. Canonical sample order

The canonical sequence order MUST be identical between Standard-Z Generator and RE Extractor.

The preferred order is:

```text
hop m' (if hopping)
    -> data spreading symbol m
        -> subcarrier n = 0..11
```

For no hopping, `m' = 0` only.

The canonical sample index is:

$$
q = n + 12m + 12N_{SF,m'}m'
$$

for the hop-major representation, with the second term generalized for the actual hop partition.

More explicitly, the implementation SHALL maintain a sample counter that increments once per extracted data RE. `q=0` is the first PUCCH data RE and `q=N_RE,data-1` is the last.

## 8. Standard-Z Generator functional behavior

For each active candidate `k`:

```text
1. map candidate bits to d_k;
2. derive u and v;
3. derive alpha for each relevant PUCCH data symbol;
4. obtain phi_u(n);
5. form reference phase theta_ref;
6. construct r_uv^(alpha,delta)(n);
7. multiply by d_k;
8. apply OCC w_i(m);
9. emit z_std[k][q].
```

### Phase-domain implementation

The Standard-Z Generator MAY use:

```text
phi_u LUT
   +
cyclic-shift phase accumulator
   -> theta_ref code
   -> sin/cos or CORDIC
   -> r_uv I/Q
```

The phase accumulator step is:

$$
\theta_{cs}[n+1]=\mathrm{wrap}(\theta_{cs}[n]+\alpha_{code})
$$

This is an implementation optimization; the normative output remains the complex sequence defined above.

### Candidate generation options

The implementation MAY:

- generate all candidates in parallel;
- generate candidates sequentially and buffer them;
- exploit the fact that candidates differ only in `d_k`.

The chosen architecture MUST expose the same logical `Z_std[k][q]` contract.

## 9. Equalized-grid input model

For a valid PUCCH data RE `q`, the grid input is:

$$
z_{noisy}[q]=z_{true}[q]+e[q]
$$

where `e[q]` represents receiver noise and residual equalization/channel error.

For ideal equalization and no noise:

$$
z_{noisy}[q]=z_{std,k_{true}}[q]
$$

This equality is the primary noise-free functional invariant.

The equalizer contract MUST preserve amplitude and phase convention compatible with the Standard-Z Generator. No additional reference derotation is assumed in the PUCCH detector.

## 10. RE extraction

RE extraction MUST:

- select the configured PUCCH PRB;
- apply the configured start symbol and length;
- handle intra-slot frequency hopping when enabled;
- skip DM-RS symbols;
- skip non-data/guard positions;
- output only PUCCH data REs;
- present samples in the canonical order defined in Section 7.

Each output sample is:

```text
z_noisy_i[15:0]
z_noisy_q[15:0]
z_noisy_valid
z_noisy_first
z_noisy_last
sample_q
```

## 11. MLE functional algorithm

For `K` valid candidates, where `K=2` for 1-bit and `K=4` for 2-bit:

$$
E_k = \sum_{q=0}^{N_{RE,data}-1}
\left|z_{noisy}[q]-z_{std,k}[q]\right|^2
$$

with:

$$
|a+jb|^2=a^2+b^2
$$

Therefore in I/Q form:

$$
E_k=\sum_q
\left(z_{noisy,I}[q]-z_{std,k,I}[q]\right)^2
+
\left(z_{noisy,Q}[q]-z_{std,k,Q}[q]\right)^2
$$

Decision:

$$
k_{best}=\arg\min_k E_k
$$

The selected candidate index maps directly to UCI bits.

## 12. Why no derotation/OCC/accumulation is required on the noisy path

The Standard-Z Generator has already constructed the full ideal sequence including:

- modulation symbol;
- low-PAPR reference sequence;
- cyclic shift;
- block-wise spreading/OCC.

The RE Extractor obtains the same physical samples from the equalized grid.

Therefore MLE compares the physical-domain sequences directly. Applying reference conjugation and OCC conjugation to only the noisy branch would change the representation being compared and would no longer be a direct sequence-to-sequence detector.

## 13. Equivalent correlation optimization

Because every ideal candidate has the same per-RE magnitude in the floating-point mathematical model, the Euclidean metric can be rearranged into a correlation form.

However, the **normative golden metric is Euclidean error `E_k`**.

A hardware optimization MAY use an algebraically equivalent correlation metric only after fixed-point scaling and candidate-norm effects are explicitly verified.

## 14. Baseline numerical convention

Input/output complex samples:

```text
I/Q = signed Q1.15, 16 bits
```

The Standard-Z Generator MAY use wider internal precision.

The MLE internal accumulator width is implementation-defined in this architecture and MUST be derived from:

- maximum number of data REs;
- maximum candidate sample magnitude;
- squaring growth;
- candidate count and comparator range;
- rounding/saturation policy.

The previous 26-bit coherent accumulator requirement MUST NOT be copied blindly into this architecture because there is no longer a linear complex accumulation stage.
