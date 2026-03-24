# intro_to_spatial_proteomics

Phase 1: TMA Dearraying
Goal: Split the full TMA image into individual core images, one file per core.
Why: Each core is a separate biological sample (different patient, different tissue site). They must be normalised and analysed independently to avoid cross-contamination of intensity distributions.

 1.1 — Export the channel/panel configuration

Output: channel_config.yaml listing all marker names in channel order
Method: Extract from image OME-XML metadata, or write manually from your panel spreadsheet
Verify: Channel count in YAML matches channel count in image


 1.2 — Dearray the TMA

Input: Full TMA image + core diameter estimate + grid dimensions (rows × cols)
Method: tma_dearray.py (blob detection on DAPI channel to find cores, grid estimation, cropping)
Output: One .ome.tif per core in a cores/ directory + core_manifest.csv
Check: Open 2–3 core TIFs in FIJI/napari to verify they look right (correct cropping, no overlap, channels intact)


 1.3 — Handle annotations (if applicable)

If a pathologist has drawn ROIs in QuPath: export them as GeoJSON first, then pass --geojson to the dearrayer
The dearrayer translates annotation coordinates to per-core local pixel space automatically
Output: Per-core .geojson files alongside the TIFs
These are used after the pipeline runs, not during — they're a spatial filter on the final cell table


 1.4 — Review the manifest

Open core_manifest.csv — check that all expected cores are present
Cores marked is_missing=True were not detected (empty position, damaged core, or detection failure)
Decide which cores to run through the pipeline (exclude obviously damaged ones)




Phase 2: Per-Core Quality Control (Pre-Processing)
Goal: Flag problematic cores before investing compute time in the full pipeline.
Run once per core.

 2.1 — QC Pre-flight

Input: Individual core TIF + panel YAML
Checks performed:

DAPI tissue coverage (is there actually tissue in this core?)
Per-channel intensity statistics (p01, p50, p99)
Zero-fraction per channel (is any critical marker completely blank?)
Saturation fraction (is any marker blown out?)


Output: QC_PRE.json with pass_qc: true/false and any critical/warning flags
If critical flags: Do not proceed with this core. Investigate why (staining failure, imaging artifact, wrong panel mapping)
Important for TMA: Use the --tma-mode flag — standard whole-slide QC thresholds are too strict for small circular cores




Phase 3: Image Ingestion & Data Structuring
Goal: Convert the raw TIF into a standardised format that all downstream modules can consume.
Run once per core.

 3.1 — Ingest

Input: Core TIF + panel YAML
What it does:

Loads the TIFF and determines axis order (CYX)
Extracts/validates channel names from metadata or panel YAML
Extracts pixel size from metadata
Computes a SHA256 hash of the source file (for reproducibility)


Output: IngestContract.json + normalised CYX numpy array on disk
Check: is_high_confidence: true means axes and channels matched metadata. If false, review the warnings


 3.2 — SpatialData conversion

Input: Ingested array + panel YAML
What it does:

Wraps the image into a SpatialData .zarr store
Sets channel coordinate names (critical — these propagate to all downstream modules)
Attaches pixel size as a spatial transformation


Output: <sample_id>.sdata.zarr
Critical flag: Must use the --panel flag so channel names flow through correctly. Silent failures occur without this
Check: Open the .zarr in Python and verify sdata.images["image"].coords["c"].values shows your marker names, not generic ch_0, ch_1




Phase 4: Background Normalisation
Goal: Correct for autofluorescence and uneven staining so that marker intensities reflect true signal.
Run once per core. Do NOT normalise across cores at this stage.

 4.1 — Background normalisation

Input: SpatialData .zarr
What it does:

Fits a Gaussian Mixture Model (GMM) per channel to separate signal from background
Computes upper-quartile normalisation values
Generates a tissue validity mask (separating tissue from empty slide)
Applies median-filter-based background subtraction


Output: Bgnorm.json contract + corrected intensity arrays written back to SpatialData + valid_mask in the masks layer
Check severity: OK = proceed. WARNING = check the failing channels but probably fine. CRITICAL = do not proceed
Check: Look at the bgnorm_summaries.csv — are the signal components sensible? (component 0 = background, component 1 = signal for most markers)


 4.2 — Visual sanity check

If available, review the bgnorm visualisation output (before/after intensity distributions per channel)
Key things to look for:

Background peak should be near zero after correction
Signal peak should be clearly separated from background
No channels should be entirely zero (staining failure) or entirely saturated






Phase 5: Cell Segmentation
Goal: Identify individual cells in the image and draw boundaries around each one.
Run once per core.

 5.1 — Cellpose segmentation

Input: SpatialData .zarr (uses DAPI/nuclear channel for segmentation)
What it does:

Runs Cellpose deep learning model to detect cell nuclei
Produces a label mask where each cell gets a unique integer ID


Key parameter: cell diameter in pixels. For typical TMA data at 0.5 µm/pixel, nuclei are ~10–15 µm → 20–30 pixels diameter
Output: labels.zarr (segmentation mask) + cells_table.h5ad (cell centroids and basic morphology) + SegmentationContract.json
Important: Uses the --use-mask flag to restrict segmentation to the tissue validity mask from bgnorm (avoids false detections in empty regions)


 5.2 — QC Post-segmentation

Input: SpatialData + labels + cells table
Checks:

Total cell count (too few = failed segmentation, too many = over-segmentation)
Cell density (cells per mm² — should be biologically plausible for your tissue type)
Border cell fraction (cells touching the edge of the core — these may have truncated measurements)
Critical marker coverage in detected cells


Output: QC_POST.json with pass/fail
Requires: --labels-path pointing to the standalone labels.zarr file
Typical ranges for TMA: 1,000–10,000 cells per core, <10% border fraction, 500–5,000 cells/mm²




Phase 6: Cell Typing
Goal: Assign a biological identity (tumour, immune, stromal, etc.) to every detected cell.
Run once per core. Two stages: coarse then fine.

 6.1 — Coarse cell typing

Input: Cells table (h5ad) with mean intensities per cell
What it does:

Applies Otsu thresholding on PanCK and CD45 to separate cells into three compartments:

Tumour (PanCK+/CD45−)
Immune (CD45+/PanCK−)
Stromal (PanCK−/CD45−)


Double-positive cells (PanCK+/CD45+) are flagged for review


Output: Updated cells table with coarse_type column + coarse typing contract
Check: Tumour fraction — for TNBC, typically 10–80%. If 0% or 100%, the threshold is wrong
Check: The Otsu bimodal separation — a strong separation ratio means confident typing


 6.2 — Clustering

Input: Coarse-typed cells table
What it does:

Runs PhenoGraph or Leiden community detection on the marker expression matrix
Assigns each cell to an unsupervised cluster


Output: Updated cells table with cluster assignments
Purpose: Provides an independent grouping that fine typing can use for validation and ambiguity resolution


 6.3 — Fine cell typing

Input: Coarse-typed + clustered cells table
What it does:

Within each compartment (Tumour, Immune, Stromal), applies rule-based marker thresholds to assign specific cell types:

Immune: CD8+ T cells, CD4+ T cells, Tregs (FOXP3+), B cells (CD20+), Myeloid (CD68+), NK cells, Dendritic cells
Tumour: Ki67+ tumour, tumour-likely, etc.
Stromal: Endothelial (CD31+), Fibroblast (SMA+)


Cells that don't meet any threshold → Untyped


Output: Final cells table with fine_type column + confidence scores + ambiguity summary
Check: Untyped fraction — ideally <20%. High untyped means thresholds may need adjustment
Check: Cell type proportions — do they make biological sense for your tissue?




Phase 7: Spatial Analysis
Goal: Quantify the spatial organisation of cell types — who is near whom, and what patterns emerge.
Run once per core.

 7.1 — Spatial metrics computation

Input: Fine-typed cells table with centroids
What it computes:

Cell type fractions — proportion of each cell type in the core
Nearest-neighbour distances — how close are immune cells to tumour cells on average?
Interaction enrichment — are certain cell type pairs found together more (or less) often than expected by chance? (Squidpy-based)
Spatial clustering scores — are immune cells aggregated in hotspots or dispersed?
Key ratios — CD4:CD8 ratio, immune:tumour ratio, etc.


Output: Spatial metrics JSON + per-core summary statistics
These metrics become the features for downstream biomarker discovery


 7.2 — Visualisation

Generates overlay plots: cell type maps, spatial interaction heatmaps, marker intensity maps
Output: PNG figures per core
Purpose: Visual validation that the computational results match what you see in the image




Phase 8: Cross-Cohort Integration (Post-Pipeline)
Goal: Combine per-core results into a single analysis-ready dataset.
Run once for the whole cohort after all cores are processed.

 8.1 — Concatenate per-core AnnData objects

Load each core's final typed cells table
Add metadata columns: core_id, patient_id, tissue_site (primary vs metastatic), clinical variables
Concatenate into one combined AnnData


 8.2 — Cross-core normalisation (for biomarker discovery)

Per-core bgnorm makes each core internally consistent, but intensities are not directly comparable across cores
Apply quantile normalisation across all cells per marker to put everything on the same scale
Store in adata.layers["cross_normalised"]


 8.3 — Apply ROI annotations (if applicable)

Load per-core GeoJSON files
For each cell, test if its centroid falls within any annotated ROI (point-in-polygon)
Add in_roi and roi_name columns to the combined table
Enables comparison between pathologist-identified regions and the rest of the core


 8.4 — Build feature matrix for biomarker discovery

Aggregate per-core spatial metrics into a patient-level feature matrix
Features per patient: cell type fractions, spatial scores, key ratios, mean intensities per cell type
This is the input for classification, survival analysis, clustering, or any other statistical modelling
