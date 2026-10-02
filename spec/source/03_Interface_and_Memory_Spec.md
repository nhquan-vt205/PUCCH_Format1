# PUCCH Format 1 Receiver — Interface and Memory Specification

Version: 1.0  
Status: Interface contract freeze

## 1. Purpose

Freeze the external control/data interfaces and internal stream contracts needed for RTL design.

The top level has two independent planes:

```text
CONTROL PLANE : AXI4-Lite -> Config RAM -> status/result
DATA PLANE    : AXI4-Full  -> RG RAM     -> RE Extractor
```

## 2. Clock/reset

Baseline:

```systemverilog
input logic clk;
input logic rst_n;
```

All blocks use the same clock domain in the baseline architecture.

Reset MUST clear:

- active and shadow configuration;
- FSM state;
- valid/last pipeline state;
- MLE accumulators/metrics;
- result/status flags.

## 3. AXI4-Lite control interface

The top-level MUST expose a standard AXI4-Lite slave interface.

Exact AXI signal widths are implementation-defined, but the baseline SHOULD use:

- 32-bit address;
- 32-bit write/read data;
- 4 byte strobes;
- one clock domain;
- normal AXI4-Lite handshake semantics.

The AXI interface itself MUST remain a conventional AXI4-Lite slave; the internal configuration storage is the `Config RAM`.

## 4. Configuration register model

The following logical registers are required. Exact addresses may be changed by RTL packaging, but field semantics MUST remain stable.

| Offset | Register | Key fields |
|---|---|---|
| 0x0000 | CTRL | enable, start, clear_result |
| 0x0004 | STATUS | busy, result_valid, error |
| 0x0010 | PUCCH_CFG0 | start_symbol, nsymb, enable |
| 0x0014 | PUCCH_CFG1 | starting_prb |
| 0x0018 | PUCCH_CFG2 | second_hop_prb |
| 0x001C | PUCCH_CFG3 | hopping, group_hopping_mode |
| 0x0020 | PUCCH_CFG4 | hopping_id |
| 0x0024 | PUCCH_CFG5 | initial_cyclic_shift, time_domain_occ |
| 0x0028 | PUCCH_CFG6 | uci_bit_count |
| 0x0030 | RESULT0 | selected candidate / UCI bits |
| 0x0034 | RESULT1 | metric_best / implementation-selected metric format |
| 0x0038 | RESULT2 | metric_second / debug |
| 0x003C | DEBUG | optional debug selector/status |

### 4.1 Required configuration fields

```systemverilog
pucch_enable
pucch_start_symbol
pucch_nsymb
starting_prb
second_hop_prb
intra_slot_hopping
group_hopping_mode
hopping_id
initial_cyclic_shift
time_domain_occ
uci_bit_count
```

These correspond to the project configuration set already established in the source design documents.

### 4.2 CTRL semantics

Recommended baseline:

```text
CTRL.enable       : enable processing
CTRL.start        : self-clearing command; accepts the current shadow config
CTRL.clear_result : clears result_valid and status/error
```

`START` MUST be rejected or ignored while `busy=1`.

## 5. Timing interface

The compute core receives:

```systemverilog
input logic [9:0] frame_idx;
input logic [4:0] slot_idx;
input logic [3:0] symbol_idx;
```

The meaning follows the current project convention:

- `frame_idx`: SFN, 0..1023;
- `slot_idx`: slot within frame for 30 kHz SCS, 0..19;
- `symbol_idx`: symbol within slot, 0..13.

## 6. AXI4-Full resource-grid interface

The top level MUST support a memory/data plane capable of loading the estimation/equalization result into RG RAM efficiently. This plane is AXI4-Full, not AXI4-Lite.

Baseline implementation profile SHOULD use:

- 32-bit data;
- 32-bit byte address;
- INCR bursts;
- 4 byte data strobes;
- one clock domain with the detector.

The AXI4-Full bus is an external transport interface. Internally, the detector SHOULD access RG RAM through a simple synchronous memory port.

## 7. RG RAM data format

Each resource-grid complex sample is:

```text
grid_word[31:0] = { I[15:0], Q[15:0] }
```

Both I and Q are signed Q1.15.

No de-rotation or OCC processing is stored in RG RAM.

## 8. Reference internal grid address model

For the baseline one-slot grid organization:

```text
slot -> symbol -> PRB -> subcarrier
```

Reference linear index:

$$
addr=((symbol\times273)+PRB)\times12+sc
$$

The maximum index for a full 14-symbol, 273-PRB slot is:

$$
14\times273\times12-1=45863
$$

Therefore a compact internal index requires 16 bits.

The external AXI address is independent of this internal index.

The actual BRAM/SRAM banking and physical memory layout are implementation-defined.

## 9. RE Extractor memory contract

The RE Extractor SHOULD use a simple internal read channel such as:

```systemverilog
output logic        grid_rd_en;
output logic [15:0] grid_rd_addr;
input  logic [31:0] grid_rd_data;
input  logic        grid_rd_valid;
```

The previous 15-bit address suggestion is insufficient for the full reference one-slot address range and MUST NOT be carried forward unchanged.

## 10. RE Extractor output contract

```systemverilog
output logic signed [15:0] noisy_i;
output logic signed [15:0] noisy_q;
output logic               noisy_valid;
output logic               noisy_first;
output logic               noisy_last;
output logic [15:0]        sample_q;
```

Semantics:

- `noisy_valid=1`: current grid word is a PUCCH data RE.
- `noisy_first=1`: first valid data RE of the PUCCH occasion.
- `noisy_last=1`: last valid data RE of the PUCCH occasion.
- `sample_q`: canonical sequence index.

The extractor MUST NOT emit DM-RS REs as valid noisy samples.

## 11. Standard-Z Generator output contract

The generator MUST provide a logically indexed candidate sequence:

```systemverilog
output logic signed [W-1:0] zstd_i;
output logic signed [W-1:0] zstd_q;
output logic                  zstd_valid;
output logic [KIDX_W-1:0]     candidate_idx;
output logic [SAMPLE_W-1:0]   sample_q;
output logic                  zstd_last;
```

`W` and `KIDX_W` are implementation-defined.

The logical output ordering MUST match the RE Extractor ordering.

## 12. MLE interface

The MLE MUST have access to matching candidate and noisy samples. Two implementation styles are allowed:

### Streaming style

```text
standard candidate sample k/q ----+
                                   +--> metric engine
noisy sample q --------------------+
```

### Buffered style

```text
Z_std buffers + Z_noisy buffer -> metric engine -> comparator
```

The external functional result is identical.

## 13. Status/result interface

Required top-level status:

```systemverilog
output logic        busy;
output logic        result_valid;
output logic [1:0]  uci_bits;
output logic        error;
```

Debug/result metrics SHOULD also be exposed:

```systemverilog
output logic [METRIC_W-1:0] metric_best;
output logic [METRIC_W-1:0] metric_second;
```

Exact metric width MUST be frozen after fixed-point analysis.
