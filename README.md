# intro_to_spatial_proteomics

## Phase 0 — Know your data before touching it

- [ ] Confirm raw image file is intact and openable
- [ ] Know your panel: which markers, which is nuclear stain, which are lineage markers
- [ ] Know your TMA layout: rows, columns, core diameter, which cores are which patients
- [ ] Know your pixel size (µm/pixel) — everything spatial depends on this
- [ ] Know your file format — QPTIFF, OME-TIFF, ImageJ TIFF — because metadata extraction differs

## Phase 1 — Dearraying

- [ ] Detect core positions in the full TMA image (blob detection, grid fitting, or manual in QuPath)
- [ ] Assign grid labels to cores (A-1, A-2, B-1...) and flag missing/damaged cores
- [ ] Crop each core to an individual image file
- [ ] Export channel/marker names as a structured panel file
- [ ] Export any pathologist annotations per core with coordinates translated to local crop space
- [ ] Visually spot-check 3–5 cropped cores to confirm correct cropping and channel order

## Phase 2 — Pre-processing QC (per core)

- [ ] Check tissue coverage — is there actually tissue in this core?
- [ ] Check per-channel intensity distributions — any blank or saturated channels?
- [ ] Check critical markers specifically (nuclear stain, key lineage markers)
- [ ] Decide pass/fail per core — don't waste compute on failed cores

## Phase 3 — Normalisation (per core, independently)

- [ ] Background subtraction — separate signal from autofluorescence per channel
- [ ] Generate a tissue validity mask (tissue vs empty slide vs debris)
- [ ] Normalise intensities within each core (e.g. upper-quartile, z-score, quantile)
- [ ] **Do not normalise across cores yet** — each core has its own staining characteristics
- [ ] Visually check before/after intensity distributions for a few channels

## Phase 4 — Segmentation (per core)

- [ ] Run cell segmentation on the nuclear channel (Cellpose, StarDist, Mesmer, or similar)
- [ ] Restrict segmentation to the tissue mask — avoids false detections in empty regions
- [ ] Check cell count: plausible for your tissue? (typical TMA core: 500–10,000 cells)
- [ ] Check cell size distribution: too many tiny fragments = over-segmentation, too few large blobs = under-segmentation
- [ ] Check border cell fraction — cells touching the crop edge have truncated measurements
- [ ] Extract per-cell measurements: mean intensity per marker, centroid coordinates, morphology (area, eccentricity)

## Phase 5 — Cell typing (per core)

- [ ] Coarse compartment assignment: tumour vs immune vs stromal (typically via lineage marker thresholding — PanCK for tumour, CD45 for immune)
- [ ] Sanity check tumour fraction — does it match what you see in the image?
- [ ] Fine cell type assignment within each compartment using marker-specific thresholds or classifiers
- [ ] Check untyped fraction — high untyped means your thresholds need adjustment or you're missing markers
- [ ] Optional: unsupervised clustering (PhenoGraph, Leiden) as an independent validation of your typed populations

## Phase 6 — Spatial analysis (per core)

- [ ] Compute cell type fractions per core
- [ ] Compute nearest-neighbour distances between cell types of interest (e.g. CD8 T cell to nearest tumour cell)
- [ ] Compute interaction enrichment scores — are cell type pairs co-located more or less than expected?
- [ ] Compute spatial organisation metrics — clustered vs dispersed immune infiltrate
- [ ] Compute biologically meaningful ratios (CD4:CD8, immune:tumour, etc.)
- [ ] If annotations exist: filter cells by ROI and compare ROI composition to whole-core composition

## Phase 7 — Cross-core integration (whole cohort)

- [ ] Concatenate all per-core cell tables into one combined table
- [ ] Add metadata: patient ID, tissue site, clinical variables
- [ ] Apply cross-core intensity normalisation now (quantile normalisation or reference-based) so cells are comparable across cores
- [ ] Build a patient-level feature matrix from per-core spatial metrics
- [ ] This is your input for biomarker discovery, classification, survival analysis, or whatever the biological question is

---

**Common failure chain:** bad channel names → wrong normalisation → wrong typing → meaningless spatial metrics. If something looks off downstream, trace it back upstream — the problem is almost always earlier than you think.
