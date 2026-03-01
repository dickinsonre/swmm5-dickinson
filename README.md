SWMM5-Dickinson
52 years of stormwater modeling in one repository.
Show Image
Show Image
Show Image
Show Image
Show Image

In 1974, I ran my first SWMM model on punch cards. Today I'm compiling the same engine to WebAssembly so it runs in a browser. This repository collects everything in between — engine enhancements, educational apps, automation scripts, and tools built across every version of SWMM from 3 to 5+.
What's Here
swmm5-dickinson/
│
├── enhancements/                        ← Patches to the EPA SWMM5 engine
│   ├── twolayer-infiltration/           ← Two-layer soil model via LID reuse
│   │   ├── src/                             lid_twolayer.h, lid_twolayer.c
│   │   ├── examples/                        Example .inp files
│   │   └── docs/                            Theory guide, OpenSWMM submission
│   ├── improved-hydraulics/             ← Surcharge, Preissmann slot, transitions
│   └── wasm-engine/                     ← SWMM5 → WebAssembly via Emscripten
│
├── apps/                                ← Interactive educational tools
│   ├── vibe-coding/                         Replit & Lovable app source
│   └── code-browser/                        SWMM5 C source explorer
│
├── scripts/                             ← Automation & conversion
│   ├── ruby-icm/                            InfoWorks ICM Ruby scripts
│   └── python-swmm/                         PySWMM & swmm_api utilities
│
└── docs/                                ← Cross-cutting documentation

Enhancements
Two-Layer Infiltration · enhancements/twolayer-infiltration/
SWMM5's three built-in infiltration methods (Horton, Green-Ampt, Curve Number) all treat the soil as a single uniform layer. Real soil profiles have distinct horizons — sandy loam over clay, topsoil over hardpan — and the interface between layers controls how fast water moves downward.
This enhancement adds two-layer soil infiltration to standard subcatchments by reusing the existing LID control's soil layer machinery and the proven percolation equation already in lid_proc.c. The approach is surgical: one new keyword (SOIL2), one new flag (UseAsInfil), roughly 39 lines touched in existing code. Every existing INP file runs identically.
ini;; Define a two-layer soil profile as a LID control
SandyLoamClay    SOIL   12.0  0.453  0.190  0.085  0.43  4.33  0.30    ;; Upper layer
SandyLoamClay    SOIL2  24.0  0.464  0.310  0.187  0.04  8.22  0.15    ;; Lower layer

;; Assign to any subcatchment with a single flag
[LID_USAGE]
S1  SandyLoamClay  0  0  0  0  0  0  *  *  0  1    ;; UseAsInfil = 1
The core percolation equation, borrowed directly from lid_proc.c :: soilFluxRates():
perc = Ksat × exp(-15.0 × (1.0 − θ/porosity))
Upper layer percolation becomes the lower layer's input. If the lower layer saturates, it throttles the upper layer. Simple, physically based, and already validated by every LID simulation ever run in SWMM5.
→ Source code · Example INP · Integration guide · Documentation

Improved Hydraulics · enhancements/improved-hydraulics/ · planned
Targeted fixes for known dynamic wave solver issues. The goal is better stability without sacrificing speed:
ProblemCurrent SWMM5 BehaviorProposed FixPreissmann slotFixed width (~1% of pipe diameter)Adaptive narrowing as pressure stabilizesCrown transitionOscillation at surcharge/free-surface boundaryDamped transition with hysteresis bandSupercritical flowAbrupt Froude limiter cutoffSmooth blending across flow regime transitionsJunction lossesContinuity-only (no head loss at junctions)Optional momentum-based head lossPond/pipe couplingInstability when storage surface area is smallRelaxation factor for stiff coupling
These are the issues I've watched modelers struggle with since SWMM3. Each fix is informed by how InfoWorks ICM handles the same problem — taking what works and adapting it for SWMM5's architecture.
→ Design notes

WASM Engine · enhancements/wasm-engine/ · in progress
EPA SWMM5 compiled to WebAssembly via Emscripten. This is what makes the vibe coding apps possible — a full SWMM5 simulation engine running client-side in any browser, no install required.
bashemcc src/*.c -o swmm5.js \
  -s MODULARIZE=1 \
  -s EXPORTED_FUNCTIONS='["_swmm_run","_swmm_open","_swmm_start","_swmm_step","_swmm_end"]' \
  -s EXPORTED_RUNTIME_METHODS='["ccall","cwrap","FS"]'
javascriptconst swmm = await SWMM5();
swmm.FS.writeFile('model.inp', inpContent);
swmm.ccall('swmm_run', 'number', ['string','string','string'],
           ['model.inp', 'report.rpt', 'output.out']);
const report = swmm.FS.readFile('report.rpt', { encoding: 'utf8' });
→ Build scripts

Apps
Vibe Coding Apps · apps/vibe-coding/
I started calling them "vibe coding apps" because the workflow is: describe what you want the hydraulics to look like, let AI generate the interface, then correct the engineering. The result is interactive educational tools that would have taken weeks to build traditionally.
Each app runs in the browser with no backend. Most are deployed on Replit or Lovable:

Infiltration Explorer — Horton, Green-Ampt, Curve Number, and Two-Layer compared side by side with real-time parameter sliders
RTK Hydrograph Generator — Interactive RDII unit hydrograph tool with R, T, K parameter exploration
Manning's Calculator — Visual open channel flow with depth, velocity, and flow area
Design Storm Generator — SCS Type I/IA/II/III distributions with custom IDF curves
Rain Garden Engine — Full LID bio-retention simulation ported from lid_proc.c
SWMM Network Builder — Drag-and-drop network construction
SWMM5 Report Reader — Parse and chart any .rpt file (up to 3,000 charts from one run)
InfoSewer to ICM App — Step-by-step migration wizard for the March 2026 ArcMap deadline

Code Browser · apps/code-browser/
SWMM5 has 40+ C source files and hundreds of functions. The code browser generates Sankey diagrams showing how data flows through the engine — from rainfall input through runoff, routing, and output. Useful for understanding what happens when you call swmm_step().

Scripts
Ruby ICM Scripts · scripts/ruby-icm/
InfoWorks ICM exposes its entire data model through a Ruby scripting API. These scripts automate the tasks I do most often:

InfoSewer → ICM conversion — Complete migration suite reading .IEDB files directly (no ArcCatalog needed). Critical for the March 2026 ArcMap discontinuation.
InfoSWMM → ICM migration — Multi-scenario import with post-import cleanup
RDII analysis — RTK parameter fitting and unit hydrograph management
Network QA/QC — Trace upstream/downstream, find disconnected nodes, compare scenarios
Batch export/import — ODEC/ODIC automation for CSV, GIS, and database formats

See also: Innovyze Open-Source-Support for the full 578-script library.
Python SWMM Scripts · scripts/python-swmm/
PySWMM and swmm_api workflows for batch simulation, parameter sweeps, and results extraction.

Timeline
This repository exists because SWMM has been a continuous thread through my entire career:
YearSWMM VersionWhat Happened1974SWMM 2First model run. Punch cards. University of Florida.1981SWMM 3 (Extran)CDM publishes the Extran manual with 7 test files I still use today1988SWMM 4Co-authored the User's Manual with Wayne Huber (EPA/600/3-88/001a)2004SWMM 5.0CRADA project with EPA. I coded the RTK/RDII implementation.2005–2023InfoSWMM18 years at Innovyze building commercial SWMM tools, RDII Analyst2023SWMM5+Joined CIMM.org Technical Advisory Committee as Chair2024ICM SWMMTransitioned to Autodesk Water, bridging SWMM5 and InfoWorks ICM2025This repoStarted collecting engine patches, WASM builds, and vibe coding apps2026NowFixathon Series. ArcMap deadline. Migration tools for the whole industry.

Related Repositories
RepositoryWhat It ContainsUSEPA/Stormwater-Management-ModelOfficial EPA SWMM5 source codeinnovyze/Open-Source-Support578 Ruby scripts + 139 SQL for InfoWorks ICM, InfoAsset, WS Propyswmm/pyswmmPython wrapper for SWMM5CIMM-ORG/SWMM5plusSWMM5+ Fortran engine with finite volume solver

Resources

Blog: swmm5.org — 1,700+ posts on SWMM, ICM, Ruby scripting, and hydraulic modeling
Newsletter: The Dickinson Canon on LinkedIn
SWMM5+ TAC: CIMM.org — Next-generation SWMM development
Community: OpenSWMM.org


Author
Robert E. Dickinson
Autodesk Water Technologist · Chair, SWMM5+ Technical Advisory Committee (CIMM.org)
52 years of continuous SWMM development. Co-author of the SWMM4 User's Manual. Coded the RTK/RDII implementation for SWMM5. 18 years building commercial SWMM tools at Innovyze. 1,700+ blog posts on swmm5.org. Currently leading migration tools and educational apps for the water infrastructure community.
License
MIT — See LICENSE.
