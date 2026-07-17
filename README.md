# HR Diffractogram from a 4D-STEM Dataset

A Jupyter notebook that detects Bragg diffraction peaks in a 4D-STEM dataset using
[py4DSTEM](https://github.com/py4dstem/py4DSTEM)'s disk-detection algorithm, then builds a
high-resolution Bragg vector histogram ("diffractogram") for an interactively selected region
of real space.

## Overview

Standard py4DSTEM workflows produce a Bragg vector map (BVM) averaged over the *entire* scan
area. This notebook extends that workflow to let you draw a polygon region of interest (ROI) on
a virtual image, discard all Bragg peaks detected outside that region, and compute a BVM from
the remaining, spatially-selected peaks only. This is useful for comparing local diffraction
signatures between different grains, phases, or features within a single 4D-STEM scan.

**Workflow**

1. Load and preprocess the 4D-STEM dataset (optional binning / log scaling)
2. Convert to a py4DSTEM `DataCube` and calibrate real & reciprocal space
3. Build a probe template (kernel) for disk detection
4. Tune Bragg-disk detection parameters on a few test diffraction patterns
5. Run disk detection over the full dataset and save the results
6. Build a Bragg vector map for the whole dataset
7. Interactively select a region of real space and compute its Bragg vector histogram
8. Export the result as TIFF

Steps 1–5 follow py4DSTEM's official disk-detection tutorial. Steps 6–8 (interactive region
selection and the regional Bragg vector histogram) are custom additions on top of that tutorial.

## Why a regional Bragg vector histogram?

A conventional way to summarize an ROI in a 4D-STEM scan is to simply average the raw diffraction
patterns over that region. This is intuitive, but it has a resolution problem: each diffraction
disk has a finite size set by the probe convergence semi-angle, and in a mixed-phase or
nanocrystalline region many grains contribute overlapping disks that sit at closely spaced but
distinct scattering angles. When you average raw intensities, those overlapping disks simply blur
into one another — the disk *shape* dominates the pattern, and closely spaced reflections from
different grains or phases become indistinguishable blobs.

Disk detection sidesteps this. For every individual diffraction pattern, py4DSTEM finds the
sub-pixel *center* of each Bragg disk by cross-correlation with a probe template, rather than
reporting the disk's raw intensity profile. Reducing every detected disk to a single point this
way removes the disk's finite size and shape from the picture. The Bragg vector histogram is then
built by accumulating these point positions, from every diffraction pattern in the selected ROI,
into a 2D histogram in reciprocal space. The result is effectively a scatter density map of "where
Bragg peaks occur," rather than an intensity image — so reflections from different grains or
phases that would overlap and merge in an averaged raw pattern can appear as separated peaks in
the histogram, because each contributing pattern's peak position is registered independently
before being combined. In that sense, the histogram trades literal diffracted intensity for a
sharper, effectively higher-resolution view of *where in reciprocal space* scattering is
occurring — which is consistent with what py4DSTEM's own benchmark dataset shows: the Bragg vector
map of a single-crystal sub-region resolves sharp, well-separated reciprocal lattice peaks that are
not resolvable in the same way from the raw averaged pattern of a larger, multi-grain region.

It's important to be clear about what this plot is and isn't. The histogram is **not** a physical
diffraction pattern — pixel values represent counts of detected peaks (optionally weighted by
correlation intensity), not scattered-electron intensity or structure factor amplitude. A strong
peak in the histogram mainly means many scan positions in the ROI registered a Bragg disk near
that reciprocal-space location, not that the corresponding reflection is intrinsically "strong" in
the usual diffraction sense. With that caveat in mind, it's a very effective tool for a specific,
practical question: *"which crystallographic spacings/phases are present in this region?"* Because
the histogram behaves like a synthetic, spatially-selected powder pattern, radially averaging it
(azimuthally integrating around the pattern center) collapses it into a 1D profile of peak density
vs. scattering-vector magnitude — directly comparable to reference *d*-spacings for candidate
phases. This makes it possible to fingerprint and disentangle overlapping phases or grain
populations within a chosen region, even when no single raw diffraction pattern in that region is
clean enough to index on its own.

## Acknowledgment

The peak-detection portion of this notebook (dataset loading, calibration, probe kernel
construction, and disk detection) is adapted from the official
[py4DSTEM tutorial notebooks](https://github.com/py4dstem/py4DSTEM_tutorials/tree/main/notebooks),
in particular the Bragg disk detection tutorial. Full credit for that part of the workflow and
for the underlying algorithms goes to the py4DSTEM development team. See the
[py4DSTEM documentation](https://py4dstem.readthedocs.io/en/latest/) for details on the disk
detection method and its parameters.

If you use py4DSTEM in published work, please cite it — see
[Citing py4DSTEM](https://py4dstem.readthedocs.io/en/latest/index.html) for the reference.

## Requirements

Developed and tested with:

- Python ≥ 3.10
- [py4DSTEM](https://github.com/py4dstem/py4DSTEM) == 0.14.18
- [HyperSpy](https://hyperspy.org/) (for loading `.hspy` files and quick interactive visualization)
- numpy
- matplotlib (with the `ipympl` backend, for `%matplotlib widget`)
- ipywidgets
- tifffile

Install into a fresh conda environment, e.g.:

```bash
conda create -n stem4d python=3.10
conda activate stem4d
pip install py4DSTEM==0.14.18 hyperspy ipympl ipywidgets tifffile matplotlib numpy
```

> py4DSTEM's own [installation guide](https://py4dstem.readthedocs.io/en/latest/) has more detail,
> including conda-forge instructions and GPU (CUDA) setup for accelerated disk detection.

## Input data

The notebook expects a 4D-STEM dataset saved as a HyperSpy `.hspy` file, with data shaped
`(Rx, Ry, Qx, Qy)` (two real-space scan dimensions, two diffraction-pattern dimensions). Adjust
the "Load the dataset" cell to point at your file. Other py4DSTEM-supported formats (raw `.dm4`,
`.emd`, `.mib`, etc.) can be substituted by changing the loading cell — see
[py4DSTEM's file I/O documentation](https://py4dstem.readthedocs.io/en/latest/) for supported
formats.

## Usage

1. Open the notebook in Jupyter Lab/Notebook (the interactive polygon-selection widget requires
   the `ipympl`/`%matplotlib widget` backend, not the inline backend).
2. Update the file paths in the **Load the dataset** cell.
3. Set the real-space and reciprocal-space calibration values (`probe_step_size_Ang`,
   `pixel_size_inv_Ang`) and the diffraction pattern origin (`qx0`, `qy0`) for your dataset.
4. Choose **either** Option A (probe extracted from a vacuum region of the dataset) **or**
   Option B (synthetic probe) to build the probe kernel — run only one of the two.
5. Tune the Bragg-disk detection parameters (`detect_params`) on the test diffraction patterns,
   then run detection on the full dataset. This step can be slow for large datasets; a CUDA GPU
   can be enabled via the `CUDA` flag in `detect_params`.
6. Run the **Interactively select a region of interest** cell, click **"1. Select ROI"**, then
   draw a polygon on the virtual image:
   - Left-click to add a vertex
   - Right-click to close the polygon
   - Press Enter to confirm
7. Run the following cell to compute and display the regional Bragg vector histogram.
8. Run the final cell to export the Bragg vector map (linear and log-scaled) as TIFF.

## Outputs

- `<filename>.braggdisks_raw.h5` — detected Bragg disk positions for the whole dataset
  (py4DSTEM `BraggVectors`, saved via `py4DSTEM.save`)
- `<filename>.detect_params.txt` — the disk-detection parameters used, saved alongside the
  Bragg disks for traceability
- `bragg_vector_map.tiff` — regional Bragg vector map, linear intensity
- `bragg_vector_map_log.tiff` — regional Bragg vector map, log-scaled intensity

## License

py4DSTEM is distributed under the [GNU GPLv3 license](https://github.com/py4dstem/py4DSTEM/blob/main/LICENSE),
and this notebook builds directly on py4DSTEM's tutorial code. Accordingly, this repository is
also released under the [GNU GPLv3 license](LICENSE).
