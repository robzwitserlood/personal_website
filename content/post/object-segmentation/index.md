---
title: Object segmentation on aerial images
draft: false
date: 2021-01-01
---

## Introduction: The Urban Imperviousness Puzzle

Cities are getting hotter and wetter. Urban planners and climate scientists know that impervious surfaces—roads, rooftops, parking lots—are a big part of the problem. But there's no complete dataset that tells us which surfaces in a city are truly impervious. So, can we use aerial imagery to fill this gap using machine learning? That's the question we set out to answer.

---

## Data Pipeline

### Acquisition

Data came from three different sources, each accessed via a different protocol:

- **BGT ground truth** (Dutch Basisregistratie Grootschalige Topografie) — retrieved via a two-step HTTP API: POST a spatial query to receive a `downloadRequestId`, poll until status is `COMPLETED`, then fetch the download link.
- **Administrative boundaries** — fetched via WFS (Web Feature Service) from PDOK's `bestuurlijkegebieden` and `wijkenbuurten` layers, returned as GML.
- **Aerial imagery** — 2022 orthophotos at 7.5 cm resolution, downloaded via WMTS at zoom level 12 using 16 concurrent workers for parallel tile fetching.

### Spatial Data Management with PostGIS

Raw metadata lives in a PostgreSQL + PostGIS database with a schema of three spatial dimension tables:

- `dimImageFiles` — catalogs source imagery files with bounding boxes and WKT geometry representations
- `dimTiles` — stores a grid of 256×256 pixel tiles covering the target region, indexed by row/column coordinates
- `dimGroundTruth` — contains labeled ground features from the BGT layers

Spatial intersection queries match ground truth geometries to tiles via a `factTilesGroundTruth` junction table. Where BGT polygons overlap (e.g. a road running through a garden), a set difference operation resolves the conflict before rasterization:

The database runs in Docker (`postgis/postgis:16-3.4`), with health checks via `pg_isready` and schema initialization scripts in `./docker/initdb`.

### Tiling & Preprocessing

Imagery is snapped to a 1 km grid, then cut into uniform 256×256 pixel tiles aligned to the underlying GeoTIFF structure. Tiles that span multiple source images are stitched by extracting four quadrants and concatenating them into a single array. Output is LZW-compressed GeoTIFF in EPSG:28992.

<figure>
    <img src="data-prep.jpg" alt="Data preparation process" />
    <figcaption>Data preparation process: spatial querying, tiling, and rasterization</figcaption>
</figure>

### Rasterization

For each tile, PostGIS is queried for intersecting BGT features. Overlapping geometries are resolved, then the resulting polygons are rasterized into a single-band label mask:

- **0** — Impervious (roads, buildings, tunnels)
- **1** — Pervious (vegetation, water, sand)
- **2** — Unknown (transitional or partially paved surfaces)

Unknown areas are filled with label ID 2 and excluded from gradient updates during training since the objective is to predict only impervious and pervious surfaces.

---

## Dataset & Fast Loading

The tiled imagery and label masks are serialized into HDF5 files. Each file holds three datasets:

- `rgb` — input imagery (3 bands, 256×256 pixels)
- `gt` — ground truth raster (single band, uint8)
- `pred` — placeholder for model predictions at inference time

A custom class inheriting from `torch.utils.data.Dataset` wraps these files for use with PyTorch's DataLoader. Loading directly from HDF5 avoids repeatedly decoding compressed images and keeps I/O from becoming the training bottleneck.

Before training, tiles are partitioned 80/10/10 into train, validation, and test sets. The split is stratified across five dimensions — impervious percentage, pervious percentage, unknown percentage, and two entropy metrics — binned into five quantiles each. This ensures the class distribution is balanced across all three splits rather than relying on a purely random shuffle.

---

## Modeling

### Architecture

The model is a U-Net with a ResNet50 encoder initialized from ImageNet pretrained weights, built with `segmentation-models-pytorch`. The decoder follows the standard U-Net design, with a softmax output over three classes.

### Three-Phase Progressive Unfreezing

Training the full network from scratch risks corrupting the pretrained encoder weights early on. Instead, parameters are unfrozen in three phases, with the learning rate stepped down at each stage:

| Phase | Trainable components | Learning rate | Epochs |
|-------|----------------------|---------------|--------|
| 1 | Segmentation head only | 0.001 | 3 |
| 2 | Decoder + head | 0.0001 | 3 |
| 3 | Full network | 0.00001 | 3 |

A `set_parameter_requires_grad()` helper enables and disables gradient computation per module between phases. The best checkpoint is saved whenever validation F1 improves.

### Loss & Optimization

The following configuration was used across all three training phases:

- **Loss:** Weighted `CrossEntropyLoss` with class weights `[0.625, 0.375, 0.0]`. Setting the unknown class weight to zero means those pixels contribute no gradient signal — the model learns only from confidently labeled surfaces.
- **Optimizer:** Adam
- **LR scheduler:** `ReduceLROnPlateau` — reduces the learning rate by a factor of 0.2 after one epoch without improvement in macro F1 (threshold 0.01).
- **Evaluation metric:** Macro F1 on classes 0 and 1 only (unknown surfaces excluded).

<figure>
    <img src="train-process.jpg" alt="Three-phase training process" />
    <figcaption>Three-phase progressive unfreezing: head → decoder → full network</figcaption>
</figure>

---

## Results & Surprises

### Macro Evaluation

The model achieved 96% recall — on par with recent literature. Confusion matrices for the training and validation splits are below. The unknown class is not represented in the predictions as expected given its zero class weight during training.

<table>
  <thead>
    <tr>
      <th colspan="5">Training set</th>
    </tr>
    <tr>
      <th></th><th></th>
      <th colspan="3">prediction</th>
    </tr>
    <tr>
      <th></th><th></th>
      <th>volledig verhard</th>
      <th>volledig onverhard</th>
      <th>onbekend</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="3"><strong>ground truth</strong></td>
      <td>volledig verhard</td>
      <td>96.2%</td><td>3.8%</td><td>0.0%</td>
    </tr>
    <tr>
      <td>volledig onverhard</td>
      <td>3.9%</td><td>96.1%</td><td>0.0%</td>
    </tr>
    <tr>
      <td>onbekend</td>
      <td>62.4%</td><td>37.6%</td><td>0.0%</td>
    </tr>
  </tbody>
</table>

<table>
  <thead>
    <tr>
      <th colspan="5">Validation set</th>
    </tr>
    <tr>
      <th></th><th></th>
      <th colspan="3">prediction</th>
    </tr>
    <tr>
      <th></th><th></th>
      <th>volledig verhard</th>
      <th>volledig onverhard</th>
      <th>onbekend</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="3"><strong>ground truth</strong></td>
      <td>volledig verhard</td>
      <td>95.9%</td><td>4.1%</td><td>0.0%</td>
    </tr>
    <tr>
      <td>volledig onverhard</td>
      <td>4.1%</td><td>95.9%</td><td>0.0%</td>
    </tr>
    <tr>
      <td>onbekend</td>
      <td>62.7%</td><td>37.3%</td><td>0.0%</td>
    </tr>
  </tbody>
</table>

<table>
  <thead>
    <tr>
      <th colspan="5">City center</th>
    </tr>
    <tr>
      <th></th><th></th>
      <th colspan="3">prediction</th>
    </tr>
    <tr>
      <th></th><th></th>
      <th>volledig verhard</th>
      <th>volledig onverhard</th>
      <th>onbekend</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="3"><strong>ground truth</strong></td>
      <td>volledig verhard</td>
      <td>95.9%</td><td>4.1%</td><td>0.0%</td>
    </tr>
    <tr>
      <td>volledig onverhard</td>
      <td>4.1%</td><td>95.9%</td><td>0.0%</td>
    </tr>
    <tr>
      <td>onbekend</td>
      <td>62.7%</td><td>37.3%</td><td>0.0%</td>
    </tr>
  </tbody>
</table>

Recall varied by region and class distribution — spatial consistency remains a challenge.

### Micro Evaluation

Visual inspection revealed where the model generalizes well and where it struggles. Roads and their immediate surroundings were classified reliably, and the model correctly identified non-vegetated, pervious surfaces — something a simple vegetation index cannot do. Backyards, however, showed poor performance: the mix of paving, gravel, and vegetation in private gardens is difficult to distinguish since they are not present in the training data.

<figure>
    <img src="generalization-road-surroundings.jpg" alt="Good generalization for roads and surroundings" />
    <figcaption>Good generalization for roads and surroundings (green: pervious, purple: impervious, yellow: unknown)</figcaption>
</figure>

<figure>
    <img src="generalization-backyards.jpg" alt="Poor generalization for backyards" />
    <figcaption>Poor generalization for backyards (green: pervious, purple: impervious, yellow: unknown)</figcaption>
</figure>

---

## Lessons Learned & What's Next

- The PostGIS + HDF5 split — spatial metadata in the database, bulk raster data in files — kept queries fast without sacrificing storage efficiency. That separation of concerns is worth reusing in any geo-ML project.
- Progressive unfreezing was essential. Training the head alone first let the randomly initialized classifier settle before the pretrained encoder weights were exposed to gradients.
- Generalization to public spaces, which are not present in the training data, remains the main open problem. Stratified splitting helped, but more granular region-based loss functions or attention mechanisms could push performance further in spatially heterogeneous areas.

---

## Conclusion

This project shows that with open data and modern computer vision, we can make real progress in mapping urban imperviousness. The pipeline — from multi-protocol data acquisition and PostGIS-backed spatial indexing, through stratified HDF5 dataset splitting, to progressive U-Net fine-tuning — is reproducible and extensible. The approach isn't perfect, but it's a strong foundation for future work.
