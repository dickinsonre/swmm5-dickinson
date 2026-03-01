# Two-Layer Subcatchment Infiltration via LID Control Reuse

**Status:** `v0.1.0` — Source code complete, ready for integration testing

## Quick Summary

Adds two-layer soil infiltration to standard SWMM5 subcatchments using the existing LID soil layer definitions. One new keyword (`SOIL2`), one new flag (`UseAsInfil`), ~39 lines changed in existing SWMM5 code.

## Files

| File | Description |
|------|-------------|
| `src/lid_twolayer.h` | Header — state structures, function prototypes |
| `src/lid_twolayer.c` | Implementation — Green-Ampt + LID percolation |
| `src/INTEGRATION.md` | Exact diffs for existing SWMM5 source files |
| `examples/TwoLayer_LID_Example.inp` | Working example with 3 soil profiles |
| `docs/TwoLayer_LID_Guide.docx` | Full technical guide |
| `docs/TwoLayer_OpenSWMM_Post.docx` | OpenSWMM research project submission |

## Core Equation

From `lid_proc.c :: soilFluxRates()`:
```
perc = Ksat × exp(-15.0 × (1.0 - θ / porosity))
```

See [src/INTEGRATION.md](src/INTEGRATION.md) for step-by-step integration instructions.
