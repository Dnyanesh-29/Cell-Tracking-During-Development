# 3D Cell Tracking in Fluorescence Microscopy

A pipeline for detecting and tracking cells through 3D+time fluorescence microscopy volumes of zebrafish embryos. Given a time-series of 3D image stacks, the system produces a **tracking graph**  a set of detected cell positions (nodes) linked across time (edges), including cell division events.

---

## Overview

Tracking cells through developing embryos is a core problem in quantitative biology. This pipeline combines a learned detection model with integer linear programming (ILP) to produce globally consistent cell tracks across hundreds of timepoints and tens of thousands of cells.

**Input:** 3D+time image volumes in Zarr v3 format `(T, Z, Y, X)`, `uint16`

**Output:** A CSV tracking graph with node positions and temporal edges

**Biological system:** Zebrafish embryo light-sheet fluorescence microscopy

<img width="2220" height="773" alt="02_mip_44b6_0113de3b" src="https://github.com/user-attachments/assets/fc4f2a1a-9192-4173-9e6e-364a18c9e6be" />


---

## Architecture

```
Raw 3D+t volume
       │
       ▼
  UNet detector
  (per-frame 3D)
       │
       ▼
Node transformer
(cross-frame edge
   probabilities)
       │
       ▼
  ILP tracker
(globally optimal
 assignment)
       │
       ▼
 Post-processing
(gap closure, division
 recovery, smoothing)
       │
       ▼
  Tracking graph
  (nodes + edges)
```

### Detection — UNet
A 3D UNet processes each timepoint independently and outputs a probability map over candidate cell centers. Only candidates above a detection threshold are kept as nodes.

### Edge prediction — Node transformer
A transformer network takes pairs of candidate nodes from adjacent frames and predicts the probability that they represent the same cell across time.

### Tracking — ILP
An integer linear program selects the globally consistent set of nodes and edges that maximises total edge probability minus penalties for cell appearance, disappearance, and division. This step jointly resolves all ambiguities across the full time series.

### Post-processing
After ILP solving, a cascade of graph operations further improves track quality:

| Step | What it does |
|------|-------------|
| Motion relinking | Re-links nodes using velocity-aware Hungarian matching weighted by learned edge probabilities |
| Gap closure | Inserts synthetic nodes to bridge 1–2 frame detection gaps, with sub-voxel position refinement via intensity centroid |
| Gap-2 recovery | Recovers longer breaks using trajectory context (direction consistency check) |
| Safe division recovery | Adds missed daughter edges using geometry constraints (parent–daughter and sister distances) |
| Linefit smoothing | Smooths interior track positions along local linear trajectory fits |
| Short track pruning | Removes isolated tracklets below a minimum length threshold |

---

## Output Format

The output CSV has one row per node or edge:

```
id, dataset, row_type, node_id, t, z, y, x, source_id, target_id
```

**Node row** (`row_type = node`): detected cell at timepoint `t`, position `(z, y, x)` in voxels. `source_id` and `target_id` are `-1`.

**Edge row** (`row_type = edge`): temporal link from `source_id` to `target_id`. All coordinate fields are `-1`. A node with two outgoing edges is a **division event** (mother cell → two daughters).

---

## Physical Scale

The default voxel scale is `z=1.625, y=0.40625, x=0.40625 µm/voxel`, reflecting the anisotropic resolution of light-sheet microscopy (Z is ~4× coarser than XY). All distance thresholds throughout the pipeline are specified in physical micrometres and converted to voxels internally.

---

## Configuration

The notebook exposes a **preset system** via environment variables. Three presets are provided:

| Preset | Description |
|--------|-------------|
| `stable` | Conservative parameters, prioritises precision |
| `score_push` | Balanced precision/recall with gap-2 recovery enabled |
| `aggressive` | Maximum recall — lower detection threshold, wider gap closure, more division recovery |

Set `BIOHUB_PRESET=aggressive` (default) or override any individual parameter:

```bash
export BIOHUB_PRESET=aggressive
export BIOHUB_DET_THRESHOLD=0.985      # detection confidence cutoff
export BIOHUB_ILP_DIVISION_WEIGHT=0.7  # ILP cost for division hypothesis
export BIOHUB_GAP_CLOSE_UM=7.0         # max µm gap to bridge
```

### Key parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `DET_THRESHOLD` | 0.985 | UNet detection confidence cutoff. Lower → more cells detected, more false positives |
| `ILP_DIVISION_WEIGHT` | 0.7 | ILP penalty for a division. Lower → more divisions accepted |
| `OUTPUT_EDGE_MAX_UM` | 15.0 | Maximum physical distance (µm) for a valid temporal edge |
| `MOTION_RELINK_TIGHT_UM` | 6.5 | Tight gate for first-pass motion relinking |
| `MOTION_RELINK_RELAXED_UM` | 11.0 | Relaxed gate for second-pass relinking |
| `GAP_CLOSE_UM` | 7.0 | Max µm displacement to close a 1-frame gap |
| `GAP2_MAX_TOTAL_UM` | 10.5 | Max µm displacement to close a 2-frame gap |
| `SAFE_DIV_MAX_UM` | 5.2 | Max parent→daughter distance for division recovery |
| `SAFE_DIV_GLOBAL_FRAC_CAP` | 0.006 | Max fraction of nodes that can be division parents |
| `OUTPUT_LINEFIT_WEIGHT` | 0.5 | Smoothing weight (0 = no smoothing, 1 = full linear fit) |

---

## Dependencies

The pipeline requires the following packages. All are available via pip or the offline wheels bundle:

| Package | Role |
|---------|------|
| `tracksdata` | Graph data structures for tracking |
| `zarr >= 3.0` | Reading Zarr v3 image volumes |
| `blosc2` | Chunk decompression |
| `geff` | Ground-truth graph format I/O |
| `ilpy` | ILP solver interface |
| `pyscipopt` | SCIP ILP backend |
| `polars` | Fast dataframe operations |
| `scikit-image` | Image processing utilities |
| `rustworkx` | Fast graph algorithms |
| `numpy`, `scipy`, `pandas` | Standard scientific stack |

---

## Repository Structure

```
biohub_aggressive.ipynb   # Main notebook (detection → tracking → CSV)
README.md                 # This file

weights/
  unet_transformer/
    split_0/
      edge_predictor_best.pth   # Trained model weights

repo/
  scripts/
    predict_unet_transformer.py  # UNet + transformer inference script
  src/                           # Model architecture source
```

---

## Results

On four held-out zebrafish embryo volumes (`T=100, Z=64, Y=256, X=256`):

| Dataset | Nodes | Edges | Divisions | Edges/Node |
|---------|-------|-------|-----------|------------|
| 44b6 (sample 1) | 26,037 | 25,168 | 52 | 0.967 |
| 44b6 (sample 2) | 47,933 | 44,768 | 187 | 0.934 |
| 6bba (sample 1) | 7,347 | 6,982 | 29 | 0.950 |
| 6bba (sample 2) | 76,517 | 74,054 | 258 | 0.968 |

**Edges/node** is the primary quality indicator — a perfect tracker on a lineage tree approaches 1.0 (every cell except those in the last frame has an outgoing link).

---

## Citation

The tracking architecture is based on **Ultrack** and the UNet+transformer approach developed at the Royer Lab, Chan Zuckerberg Biohub:

> Bragantini et al. *"Large-scale multi-hypotheses cell tracking using ultrametric contours maps."* Nature Methods, 2025.

```bibtex
@article{bragantini2025ultrack,
  title   = {Large-scale multi-hypotheses cell tracking using ultrametric contours maps},
  author  = {Bragantini, Jo{\~a}o and others},
  journal = {Nature Methods},
  year    = {2025}
}
```
