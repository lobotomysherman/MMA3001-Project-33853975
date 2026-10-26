# 2D Apple-Tree Segmentation and Skeleton Extraction

MMA3001 – Dataset 1.

A single-script pipeline that segments apple trees in paired RGB and depth images, thins the segmented trunk and branches to a one-pixel skeleton, and compares two segmentation approaches. It records accuracy (where ground truth is available), runtime and diagnostics for every run.

## Methods

Two segmentation methods are run on every observation and then passed through the same skeletonisation step.

| Method | Input | Idea |
| --- | --- | --- |
| `baseline` | RGB | Colour threshold on excess green (`2G − R − B < GREEN_THRESHOLD`) combined with a brightness window, to keep non-green, mid-brightness (wood-like) pixels. |
| `regression_depth` | Depth | Fits a linear background trend to the median depth of each "dense" image row (ordinary least squares, `np.linalg.lstsq`). Pixels that stand out above the trend by more than a fraction of the depth range are kept. The result is closed once to bridge gaps, then the 8-connected component overlapping the lower-centre seed box the most is selected as the tree. |

