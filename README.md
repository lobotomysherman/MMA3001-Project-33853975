# 2D Apple-Tree Segmentation and Skeleton Extraction


A single-script pipeline that segments apple trees in paired RGB and depth images, thins the segmented trunk and branches to a one-pixel skeleton, and compares two segmentation approaches. It records accuracy (where ground truth is available), runtime and diagnostics for every run.

## Methods

Two segmentation methods are run on every observation and then passed through the same skeletonisation step.

| Method | Input | Idea |
| --- | --- | --- |
| `baseline` | RGB | Colour threshold on excess green (`2G − R − B < GREEN_THRESHOLD`) combined with a brightness window, to keep non-green, mid-brightness (wood-like) pixels. |
| `regression_depth` | Depth | Fits a linear background trend to the median depth of each "dense" image row (ordinary least squares, `np.linalg.lstsq`). Pixels that stand out above the trend by more than a fraction of the depth range are kept. The result is closed once to bridge gaps, then the 8-connected component overlapping the lower-centre seed box the most is selected as the tree. |

**Skeletonisation** uses a from-scratch, vectorised Zhang–Suen thinning implementation (`extract_centreline`).

**Evaluation** (only when a ground-truth mask is mapped for an observation):

- Foreground IoU against the ground-truth mask.
- Skeleton precision, recall and F1 using nearest-neighbour distances (KD-tree) with a pixel tolerance (`SKELETON_TOLERANCE`, default 3 px). The ground-truth mask is thinned with the same algorithm before comparison.

Results are averaged per tree first, then across trees, so trees with more captures do not get extra weight.

## Requirements

- Python 3.8 or newer
- `numpy`, `scipy`, `pillow`, `matplotlib` (see `requirements.txt`)
- **Tkinter**, used for the folder-selection dialogs. It ships with the standard Python installers on Windows and macOS. On Debian/Ubuntu install it with `sudo apt install python3-tk`. A graphical desktop session is required because the script opens native folder dialogs.

## Installation

```bash
git clone <your-repo-url>
cd <your-repo-name>

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

## Input data

The script discovers image pairs recursively. File names must follow this pattern (PNG, TIF or TIFF):

```
<tree>_<season>_RGB-<HH-MM-SS>.png      e.g. R10N07_Winter_RGB-12-10-12.png
<tree>_<season>_D-<HH-MM-SS>.png        e.g. R10N07_Winter_D-12-10-12.png
```

Notes:

- An observation is only processed if exactly one RGB and one depth file match its `(tree, season, timestamp)` key. Unmatched or ambiguous frames are skipped and printed to the console.
- Folders whose names contain `gt`, `ground truth` or `annotations` are ignored when discovering inputs, so annotations are never treated as images.
- The depth image must be greyscale (three identical channels are reduced to one) and the same size as the RGB image. Depth values are treated as uncalibrated intensities, not metres.
- If the folder you select contains a `Dataset/` subfolder, the script uses it for observations. If a `Dataset (With Summer GT)/` folder is also present, it reports this but does **not** load annotations automatically (see [Ground truth](#ground-truth)).

The dataset is not included in this repository.

## Usage

```bash
python Tree_Project.py
```

1. Choose the folder containing your RGB and depth images (or its parent `Dataset` folder).
2. Choose where to save results.
3. The script verifies its own algorithms on synthetic data, processes every pair, and writes a timestamped run folder.

Progress is checkpointed after each observation, so an interrupted run still leaves a usable `results.csv`.

## Configuration

All settings are constants near the top of `Tree_Project.py`.

| Setting | Default | Purpose |
| --- | --- | --- |
| `MAX_PAIRS` | `None` | Limit the number of observations processed (`None` = all). |
| `FIRST_PREVIEW` | `"R10N07_Winter_12-10-12"` | Observation processed first, if present. |
| `MAX_FIGURE_PAIRS` | `None` | Limit how many observations get comparison figures (`None` = all). |
| `SAVE_MASK_IMAGES` | `False` | Also save binary mask and skeleton PNGs under `masks/`. |
| `INVALID_DEPTH_VALUES` | `[0]` | Depth codes treated as missing. Confirm with the data provider. |
| `ROI` | `None` | Optional `(left, top, right, bottom)` pixel region; right and bottom are exclusive. |
| `GREEN_THRESHOLD` | `0.08` | Excess-green cutoff for the baseline method. |
| `DEPTH_CONTRAST` | `0.10` | Foreground threshold as a fraction of the 5th–95th percentile depth range above the row background. |
| `DEPTH_DIRECTION` | `1` | `+1` matches the supplied winter depth display; use `-1` for reversed encoding. |
| `DENSE_ROW_FRACTION` | `0.40` | Rows with more valid pixels than this are used for the background fit. |
| `SEED_BOX` | `(0.35, 0.55, 0.65, 0.88)` | Relative `x0, y0, x1, y1` box around the lower trunk, used to pick the tree component. |
| `SENSITIVITY_CONTRASTS` | `[0.06, 0.10, 0.14, 0.18]` | Depth-contrast values tried in the sensitivity sweep. |
| `SENSITIVITY_LIMIT` | `20` | Number of development observations included in the sweep. |
| `SKELETON_TOLERANCE` | `3` | Pixel tolerance when matching skeletons to ground truth. |
| `GT_FILES` | `{}` | Mapping from observation name to its ground-truth mask path. |
| `TEST_TREES` | `set()` | Tree IDs held out as the test split. |

### Ground truth

`GT_FILES` is empty by default, so no accuracy scores are produced until you fill it in. Map each observation name to a binary annotation:

```python
GT_FILES = {
    "R01N01_Summer_12-40-50": r"path/to/R01N01_Summer_12-40-50_gt.png",
}
```

Annotations must be greyscale, the same size as the RGB image, and contain only the values 0/1 or 0/255 (non-zero is wood). Depth images are **not** ground truth.

### Development and test splits

Trees listed in `TEST_TREES` are labelled `test`; all others are `development`. The sensitivity sweep runs only on development trees, so settings are never tuned on held-out data. If `TEST_TREES` is empty, the run is flagged as exploratory in `READ_RESULTS.txt`.

## Outputs

Each run creates a timestamped folder (`YYYYMMDD_HHMMSS_ffffff`) inside the results directory you choose:

| File | Contents |
| --- | --- |
| `results.csv` | Per-observation, per-method metrics: IoU, precision, recall, F1, runtime, skeleton size and depth coverage. Score columns are blank when no ground truth was supplied. |
| `summary.csv` | Per-tree-averaged metrics by split and method. |
| `sensitivity.csv` | Results of the depth-contrast sweep. |
| `regression_models.csv` | Background-fit diagnostics (status, intercept, slope, RMSE, dense-row fraction). |
| `errors.csv` | Observations that failed to process, with error messages. |
| `profiling.csv`, `timing_summary.json` | Exclusive wall-clock time per pipeline stage. |
| `settings.json` | The exact settings used, for reproducibility. |
| `checks.txt` | Outcome of the built-in synthetic verification. |
| `READ_RESULTS.txt` | Caveats to read before interpreting the numbers. |
| `comparisons/` | Three-panel PNGs per observation and method: RGB, estimated mask and skeleton overlay. |
| `masks/` | Binary mask and skeleton PNGs, only if `SAVE_MASK_IMAGES = True`. |

## Built-in verification

On every run, `verify_algorithms()` checks the implementation on synthetic data before any real image is processed. It tests recovery of known regression coefficients, trunk segmentation on a synthetic scene, handling of missing depth, connectivity and idempotence of the thinning, and perfect-match metrics. These checks verify the code, not real-tree performance.

## Limitations

- Results are two-dimensional masks and skeletons. They do not establish 3D or physical accuracy.
- The colour baseline can keep grass, wires and shadows. The depth method can keep leaves and wires that touch the trunk.
- The depth method assumes a lower-centre trunk, a roughly linear row trend and the configured depth encoding.
- Background-fit RMSE measures how well the background was modelled, not tree accuracy.
- Without ground truth, the sensitivity sweep shows how outputs change with the threshold, not which threshold is most accurate.
- Timings are single-run wall-clock measurements, not a rigorous benchmark.
- Failed observations are listed in `errors.csv` and should be reported rather than silently omitted.

## Project structure

```
.
├── Tree_Project.py     # The full pipeline (settings, I/O, segmentation, skeletonisation, evaluation, reporting)
├── requirements.txt
├── .gitignore
└── README.md
```

Inside `Tree_Project.py`, the code is organised into numbered sections: settings, data reading, tree finding, evaluation and plotting, folder selection and run setup, per-observation processing, aggregation and reports, and the entry point. Docstrings use NumPy style.
