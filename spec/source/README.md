# PUCCH Format 1 Receiver — Design-Ready Specification Set

Version: 1.0  
Purpose: freeze the intended algorithm and architecture before RTL design.

## Specification authority

This specification set supersedes the previous architecture interpretation in which `Y_hat -> derotation -> OCC -> coherent accumulation -> scalar Z -> MLE` was the primary detector path.

The intended architecture is:

```text
                 AXI4-Lite
                     |
                 Config RAM
                     |
                 Config Decode
              +------+------+
              |             |
              v             v
      Standard-Z Generator  RG RAM
              |             |
       Z_std[k][sample]    AXI4-Full
              |             |
              |          RE Extract
              |             |
              |       Z_noisy[sample]
              +------+------+
                     |
                    MLE
                     |
                  UCI bits
```

## Documents

1. `01_System_Architecture_Spec.md` — block ownership, dataflow, operating modes, architectural boundaries.
2. `02_Algorithm_and_Reference_Model_Spec.md` — 3GPP-derived functional algorithm and exact candidate sequence definition.
3. `03_Interface_and_Memory_Spec.md` — AXI4-Lite register model, AXI4-Full resource-grid interface, internal stream contracts.
4. `04_MLE_and_DataPath_Spec.md` — `Z_std`/`Z_noisy` representation, MLE metric, ordering, widths, implementation constraints.
5. `05_Verification_and_Design_Constraints.md` — golden model, assertions, test plan, acceptance criteria and resolved ambiguities.

## Normative language

- **MUST**: mandatory architecture/functional requirement.
- **SHOULD**: recommended unless a documented implementation reason exists.
- **MAY**: implementation option that does not change the functional contract.

## Frozen conceptual terminology

- `z_std[k][q]`: ideal PUCCH data sequence for candidate `k` at canonical sample `q`.
- `Z_std[k]`: architectural name for the candidate standard sequence/vector; it is **not** a scalar coherent statistic.
- `z_noisy[q]`: equalized resource-grid sample selected by RE extraction.
- `Z_noisy`: architectural name for the complete extracted noisy sequence/vector; it is **not** a scalar coherent statistic.
- `metric[k]`: MLE comparison metric between `Z_noisy` and `Z_std[k]`.

The existing algorithm documents used `z(...)` for the transmitted per-RE sequence and `Z` for a coherent-combining scalar. This specification set deliberately separates those concepts to remove the ambiguity.
