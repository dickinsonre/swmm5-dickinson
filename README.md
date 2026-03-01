# SWMM5-Dickinson

**52 years of stormwater modeling — engine enhancements for EPA SWMM5.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![SWMM Version](https://img.shields.io/badge/SWMM-5.2%2B-green.svg)]()

---

In 1974, I ran my first SWMM model on punch cards. Today I'm compiling the same engine to WebAssembly so it runs in a browser. This repository collects engine enhancements built across every version of SWMM from 3 to 5+.

## What's Here

```
swmm5-dickinson/
│
├── TwoLayer_LID_Subcatchments/        ← Two-layer soil infiltration enhancement
│   ├── lid_twolayer.h                 # Header — state structures, function prototypes
│   ├── lid_twolayer.c                 # Implementation — Green-Ampt + LID percolation
│   ├── TwoLayer_LID_Integration_Guide.md  # Exact changes needed in existing SWMM5 files
│   ├── TwoLayer_LID_Example.inp       # Complete working example with subcatchments
│   └── TwoLayer_LID_Guide.docx        # Full technical guide (Word)
│
├── README.md
└── LICENSE
```

## Two-Layer Infiltration

SWMM5's three built-in infiltration methods (Horton, Green-Ampt, Curve Number) all treat the soil as a single uniform layer. Real soil profiles have distinct horizons — sandy loam over clay, topsoil over hardpan — and the interface between layers controls how fast water moves downward.

This enhancement adds two-layer soil infiltration to standard subcatchments by reusing the existing LID control's soil layer machinery and the proven percolation equation already in `lid_proc.c`. The approach is surgical: one new keyword (`SOIL2`), one new flag (`UseAsInfil`), roughly 39 lines touched in existing code. Every existing INP file runs identically.

```
                    Rainfall / Runon
                         │
                         ▼
    ┌─────────────────────────────────────────┐
    │         UPPER SOIL LAYER                │  ← LID SOIL line
    │   (Green-Ampt infiltration from         │    (thickness, porosity, FC,
    │    surface; LID percolation out)         │     WP, Ksat, suction, IMD)
    ├─────────────────────────────────────────┤
    │    perc = Ksat × exp(-15×(1-θ/n))       │  ← LID percolation equation
    ├─────────────────────────────────────────┤
    │         LOWER SOIL LAYER                │  ← LID SOIL2 line (NEW)
    │   (Receives percolation from upper;     │    (same 7 parameters)
    │    LID percolation to deep GW)          │
    └─────────────────────────────────────────┘
                         │
                         ▼
                  Deep Percolation
                  (→ Groundwater)
```

### The Key Insight

The LID module already has a sophisticated soil moisture model with full parameter parsing, validation, and unit conversion. Instead of building a new infiltration method from scratch (14 new parameters, new INP format, hundreds of lines of new parsing code), this approach:

1. Adds **`SOIL2`** keyword to `[LID_CONTROLS]` — identical 7 parameters as existing `SOIL`
2. Adds **`UseAsInfil`** flag to `[LID_USAGE]` — set to `1` to override default infiltration

That's it. ~39 lines of changes to existing SWMM5 source code.

| Aspect | Traditional Approach | This Approach |
|--------|---------------------|---------------|
| New INP parameters per subcatch | 14 | **1** |
| New INP section format | Yes | **No** |
| New `[OPTIONS]` keyword | Yes | **No** |
| Reuses existing parser | No | **Yes** |
| Backward compatible | Breaks old files | **100%** |
| Per-subcatchment control | All or nothing | **Per-subcatchment** |

### Quick Start

```ini
;; 1. Define a soil profile as a LID control
[LID_CONTROLS]
SandyLoamClay    BC
SandyLoamClay    SURFACE  0  0  0  0  0
SandyLoamClay    SOIL     12.0  0.453  0.190  0.085  0.43   4.33  0.30   ;; Upper
SandyLoamClay    SOIL2    24.0  0.464  0.310  0.187  0.04   8.22  0.15   ;; Lower (NEW)
SandyLoamClay    STORAGE  0  0  0  0
SandyLoamClay    DRAIN    0  0  0  0  0  0

;; 2. Assign to subcatchments — one flag does it all
[LID_USAGE]
;;Subcatch  LID            Num Area Width Sat FrI ToP Rpt Drn FrP UseAsInfil
S1          SandyLoamClay  0   0    0     0   0   0   *   *   0   1
S2          SandyLoamClay  0   0    0     0   0   0   *   *   0   1
;; S3 uses standard [INFILTRATION] — no LID override
```

- **S1 and S2**: Pervious area infiltration uses SOIL (upper) + SOIL2 (lower) with Green-Ampt surface infiltration and LID exponential percolation between layers
- **S3**: Continues using the standard `[INFILTRATION]` method (Green-Ampt, Horton, etc.)
- **Existing INP files**: Work identically — `SOIL2` and `UseAsInfil` are optional

### How to Integrate

This is an alternate engine enhancement — it patches into the existing SWMM5 source tree. See [`TwoLayer_LID_Integration_Guide.md`](TwoLayer_LID_Subcatchments/TwoLayer_LID_Integration_Guide.md) for the exact changes needed in:

| File | Change | Lines |
|------|--------|-------|
| `lid.h` | Add `soil2` + `hasSoil2` to `TLidProc` | ~3 |
| `lid.c` | Parse `SOIL2` keyword (copy of `SOIL` case) | ~15 |
| `lid.c` | Parse `UseAsInfil` in `[LID_USAGE]` | ~12 |
| `subcatch.c` | `if/else` to call `lidTwoLayer_getInfil` | ~5 |
| `project.c` | Init and close calls | ~4 |
| **Total existing code** | | **~39 lines** |

Plus the two new files: [`lid_twolayer.h`](TwoLayer_LID_Subcatchments/lid_twolayer.h) (~80 lines) and [`lid_twolayer.c`](TwoLayer_LID_Subcatchments/lid_twolayer.c) (~310 lines).

### The LID Percolation Equation

The core equation, borrowed directly from `lid_proc.c :: soilFluxRates()`:

```c
percRate = kSat * exp(-15.0 * (1.0 - theta / porosity));
```

| Saturation | θ/n | % of Ksat |
|-----------|-----|-----------|
| 100% | 1.0 | 100% |
| 90% | 0.9 | 22.3% |
| 80% | 0.8 | 5.0% |
| 70% | 0.7 | 1.1% |
| 50% | 0.5 | 0.055% |

Applied twice: upper→lower percolation and lower→deep percolation.

### Typical Soil Parameters

Values for both `SOIL` and `SOIL2` lines (USDA texture classes):

| Soil Type | Porosity | Field Cap | Wilt Pt | Ksat (in/hr) | Suction (in) |
|-----------|----------|-----------|---------|-------------|--------------|
| Sand | 0.437 | 0.062 | 0.024 | 4.74 | 1.93 |
| Sandy Loam | 0.453 | 0.190 | 0.085 | 0.43 | 4.33 |
| Loam | 0.463 | 0.232 | 0.116 | 0.13 | 3.50 |
| Silt Loam | 0.501 | 0.284 | 0.135 | 0.26 | 6.69 |
| Clay Loam | 0.464 | 0.310 | 0.187 | 0.04 | 8.22 |
| Clay | 0.475 | 0.378 | 0.265 | 0.01 | 12.45 |

## Timeline

| Year | SWMM Version | What Happened |
|------|-------------|---------------|
| 1974 | SWMM 2 | First model run. Punch cards. University of Florida. |
| 1981 | SWMM 3 (Extran) | CDM publishes the Extran manual with 7 test files I still use today |
| 1988 | SWMM 4 | Co-authored the User's Manual with Wayne Huber (EPA/600/3-88/001a) |
| 2004 | SWMM 5.0 | CRADA project with EPA. I coded the RTK/RDII implementation. |
| 2005–2023 | InfoSWMM | 18 years at Innovyze building commercial SWMM tools, RDII Analyst |
| 2023 | SWMM5+ | Joined CIMM.org Technical Advisory Committee as Chair |
| 2024 | ICM SWMM | Transitioned to Autodesk Water, bridging SWMM5 and InfoWorks ICM |
| 2025 | This repo | Started collecting engine patches and educational tools |
| 2026 | Now | Fixathon Series. ArcMap deadline. Migration tools for the whole industry. |

## Related Resources

| Repository | What It Contains |
|---|---|
| [USEPA/Stormwater-Management-Model](https://github.com/USEPA/Stormwater-Management-Model) | Official EPA SWMM5 source code |
| [innovyze/Open-Source-Support](https://github.com/innovyze/Open-Source-Support) | 578 Ruby scripts + 139 SQL for InfoWorks ICM, InfoAsset, WS Pro |
| [pyswmm/pyswmm](https://github.com/pyswmm/pyswmm) | Python wrapper for SWMM5 |
| [CIMM-ORG/SWMM5plus](https://github.com/CIMM-ORG/SWMM5plus) | SWMM5+ Fortran engine with finite volume solver |

- **Blog:** [swmm5.org](https://swmm5.org) — 1,700+ posts on SWMM, ICM, Ruby scripting, and hydraulic modeling
- **Newsletter:** [The Dickinson Canon](https://www.linkedin.com/in/robertdickinson/) on LinkedIn
- **SWMM5+ TAC:** [CIMM.org](https://cimm.org) — Next-generation SWMM development

## Author

**Robert E. Dickinson**
Autodesk Water Technologist · Chair, SWMM5+ Technical Advisory Committee (CIMM.org)

52 years of continuous SWMM development. Co-author of the SWMM4 User's Manual. Coded the RTK/RDII implementation for SWMM5. 18 years building commercial SWMM tools at Innovyze. 1,700+ blog posts on swmm5.org. Currently leading migration tools and educational apps for the water infrastructure community.

## License

MIT — See [LICENSE](LICENSE).
