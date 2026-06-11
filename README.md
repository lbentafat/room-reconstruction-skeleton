# Room Reconstruction and Structural Skeleton from Monocular Video

This project turns a 30-second handheld phone video of a small indoor
room into a 3D point cloud and a structural skeleton (planes,
intersection lines, corner nodes). It was built for the Humanoid intern
challenge *From Video to 3D Reconstruction*.

**Author:** Lina Bentafat

## What the pipeline produces

From a monocular phone video, the output is:

- a dense 3D reconstruction with a per-point confidence score
- a structural skeleton: dominant planes (faces), pairwise intersection
  lines (edges), and triple-plane intersections (corner nodes)
- validation tests for the main design choices (Manhattan-prior test,
  RANSAC threshold sweep, plane-fit residuals)

The challenge brief asks for a geometrically coherent 3D representation.
The skeleton is added on top of the point cloud because a compact
structural map is a more useful starting point for a humanoid robot
than a raw 3D cloud.

![Final skeleton: planes in colour, intersection lines in red, corner
nodes in black.](figures/skeleton_final.png)

## Pipeline

The notebook `notebook.ipynb` runs end-to-end on Google Colab (A100
recommended). Each cell prints its inputs, intermediate counts, and a
self-check before writing to disk.

**1. Frame selection.** From 964 video frames, 40 are kept. Three
filters run in order: an exposure gate (drop frames that are too dark
or too bright), a sharpness gate (drop the blurriest 15% by Laplacian
variance), and an arc-length sampling of the camera path. The
arc-length step accumulates optical flow magnitude between consecutive
frames and places 40 targets at equal intervals of motion. Sampling by
motion rather than by time gives uniform coverage of the room whether
the camera moved fast or slow.

**2. Dense reconstruction with MASt3R.** The 40 frames are reconstructed
using MASt3R (Matching And Stereo 3D Reconstruction, NAVER Labs Europe)
with a sliding-window pair graph of size 12. The fully-connected graph
was tried first but exceeded the Colab CPU RAM budget during global
alignment; `swin-12` reduces the pair count from 1560 to 960 while
keeping high local connectivity. Output: 5.9 M points with per-point
confidence.

**3. Cleaning.** The bottom 30% of points by MASt3R confidence are
removed. These are mostly saturated window pixels and depth
discontinuities. The cloud is then voxel-downsampled to a 300 K-point
budget. The voxel size is chosen by binary search so the downsample is
independent of MASt3R's arbitrary scene scale.

**4. Geometric confidence layer.** For each point a local 3D covariance
is computed over its 20 nearest neighbours. The three eigenvalues
describe how the patch spreads in three perpendicular directions. The
smallest eigenvalue normalised by the sum is the surface-variation
score; the per-point confidence is `1 − 3·λ_min/Σλ`. A perfectly planar
neighbourhood scores 1, an isotropic blob scores 0.

**5. Plane extraction (confidence-weighted RANSAC).** Iterative RANSAC
extracts the dominant planes one at a time. Each candidate is scored
not by inlier count but by the total geometric confidence of its
inliers, so flat well-supported regions weigh more than noisy ones.
Up to 8 planes are extracted; each plane's normal, offset, inlier
count, and mean confidence are stored.

**6. Manhattan-prior validation.** Before applying any indoor-room
prior, the pipeline tests whether the room actually behaves like a
Manhattan world. Pairwise angles between plane normals are computed;
at a strict 5° tolerance, 39% of all pairs are aligned, but the four
largest planes deviate from one of three data-driven orthogonal axes
by less than 2°. The size-weighted mean deviation is 4.24°. This is
reported as a measured property, not an assumed one.

**7. Selective Manhattan regularisation.** Only planes within 5° of a
data-driven axis are snapped to it. Planes at odd angles (furniture
surfaces, doorway frames) are left unchanged. After snapping, the
residual deviation of the snapped planes is verified to be 0.000°, and
all pairs of snapped planes are exactly 0° or 90° apart. Plane offsets
are re-anchored on their centroids; the mean offset shift is 4 mm in
MASt3R metric units.

**8. Skeleton edges.** Pairs of planes are intersected analytically.
The line direction is the cross product of the two normals; a point on
the line comes from solving the joint 3×3 system. A line is kept only
if the planes are not near-parallel (cos angle ≤ 0.96) and the cloud
contains at least 300 points within 10 cm of the line. Endpoints are
trimmed to the 0.5–99.5% range of the supporting data along the line.
12 lines survive with mean length 2.10 m.

**9. Corner nodes (triple-plane intersections).** Each triple of
planes is solved as a 3×3 linear system. Candidates with `|det N| <
0.05` are discarded as ill-conditioned. Surviving candidates must
(a) lie inside the scene bounding box expanded by 20 cm, and (b) be
supported by at least 200 cloud points within 30 cm. Near-duplicates
are fused by support-weighted clustering. Line endpoints within 40 cm
of a node are then snapped to it.

## What was validated, and how

| Decision | How it was tested | Result |
|---|---|---|
| Frame count = 40 | Memory budget on Colab A100 plus coverage | swin-12 fits in RAM with full coverage |
| swin-12 vs complete graph | Tried complete first; CPU RAM overflow | swin-12 = 960 pairs vs 1560, runs cleanly |
| MASt3R confidence threshold | Iterative tightening + downstream stability | keep top 70% (drop bottom quantile 0.30) |
| Voxel size | Binary search to a 300 K-point budget | 0.011 chosen for this scene |
| Manhattan prior | Pairwise plane-angle test at 5° + per-plane deviation from 3 data-driven axes | Holds for 4 largest planes; size-weighted mean dev = 4.24° |
| RANSAC distance threshold | Controlled sweep across 5 values (0.02 … 0.06) with pre-defined criteria | 0.04 wins on 4/5 criteria vs the 0.03 baseline |
| Line-extraction support distance | Sensitivity check at 0.05, 0.10, 0.15 | 0.10 keeps real edges and rejects phantoms |
| Corner-node detector | Switched from line–line closest-approach to triple-plane intersection after Manhattan made several lines exactly parallel | 5 nodes recovered at architectural corners |

![RANSAC threshold sweep: 0.04 wins on 4 of 5 quality criteria against
the 0.03 baseline.](figures/ransac_sweep.png)

## Results on the supplied video

A 30-second handheld video of a furnished living room (sofa, desk, two
chairs, lamps, TV, AC unit, two windows, doorway). 1080×1920 at 30 fps.

| Stage | Output | Number |
|---|---|---|
| Frames selected | exposure + sharpness + arc-length | 40 |
| MASt3R points | raw, before filtering | 5,898,240 |
| After confidence filter (drop bottom 30%) | | 4,128,768 |
| After voxel downsample | budget = 300 K | 339,556 |
| Planes extracted | covering 85% of points | 8 |
| Planes Manhattan-regularised | within 5° of an axis | 6 |
| Skeleton lines | after support and length filtering | 12 |
| Corner nodes | triple-plane intersections, physically supported | 5 |

The reconstruction has known weaknesses on this scene: the two bright
windows saturate in nearly every frame, the polished floor and glass
produce specular artifacts, and casual handheld motion leaves some
surfaces under-covered. The confidence layer is meant to make these
weaknesses explicit rather than hide them  low-confidence regions
match the saturated and reflective patches.

![Confidence heatmap on the cleaned point cloud: green = trusted
geometry, red = artifact-prone regions.](figures/confidence_heatmap.png)

## How to run

The notebook is self-contained and reproducible top-to-bottom on Colab.
Each cell declares its inputs, writes intermediate `.npz` and `.ply`
files to Google Drive, and uses plain assertions so a missing file
fails with a clear message rather than a deep traceback.

1. Open `notebook.ipynb` in Colab. Select an A100 runtime. (A T4 also
   works for everything except cell 2 if reduced to 25 frames and
   `swin-7`.)
2. Mount Google Drive. Place `challenge.mp4` under
   `/MyDrive/Humanoid_3D_Reconstruction/videos/`.
3. Run all cells. Each prints `[ok] ...` checkpoints. Total runtime on
   A100 is about 10 minutes.
4. Outputs are written to
   `/MyDrive/Humanoid_3D_Reconstruction/mast3r_output/`. The `.ply`
   files open in MeshLab, CloudCompare, or https://3dviewer.net.

For the cleanest 3D inspection, download `outputs/skeleton_final.ply`
and `outputs/clean.ply` from this repo and open them in MeshLab or
CloudCompare. The PNG figures in the README are 2D screenshots and do
not capture the depth and density of the real point clouds.

### Requirements

See `requirements.txt`. Main dependencies: PyTorch 2.x with CUDA,
MASt3R (cloned from `naver/mast3r`), Open3D, OpenCV, NumPy.

## Design notes

Some choices are not obvious from the code alone. This section explains
the reasoning behind them.

**Frame selection by motion, not time.** Time-uniform sampling wastes
frames where the camera is still and skips detail where it is moving
fast. Arc-length sampling along the cumulative motion signal gives
uniform spatial coverage of the room independently of the operator.

**MASt3R rather than DUSt3R or COLMAP.** COLMAP fails on the
textureless walls that dominate indoor scenes. DUSt3R works but its
successor MASt3R adds a feature-matching head that produces cleaner
indoor geometry, especially on weakly-textured walls. A first attempt
used DUSt3R with `swin-3` (the only graph that fit a T4); the result
had visibly warped slabs, which is what motivated switching to MASt3R
on an A100 with a wider window.

**Why a confidence layer.** Casual phone video gives noisy
reconstructions with reliable and unreliable regions. A consumer of
this output a robot, for example  needs to know which parts to
trust. The geometric confidence is computed rather than learned, so
it is explainable and controllable. It is also used as a weight inside
RANSAC, so confidence shapes the structural extraction directly rather
than only the visualisation.

**Why test the Manhattan prior before applying it.** Manhattan-world is
a common indoor prior, but applying it blindly forces furniture into
wall orientations. The pre-test at 5° tolerance shows that only the
four largest planes qualify, so only those are regularised. The
post-snap pairwise check confirms zero deviation. This turns the prior
into a measured, falsifiable claim rather than an assumption.

**Why a controlled parameter sweep, not eyeballed tuning.** The RANSAC
distance threshold has a direct mechanism for trading off plane
tightness against fragment count. Sweeping across five values with
pre-defined quality criteria (longer mean line length, higher mean
support, lower variance, fewer short fragments) makes the choice
reproducible. 0.04 wins on 4 of 5 criteria.

**Why triple plane intersections for nodes.** The first node detector
looked for pairs of lines that came close. After Manhattan
regularisation many lines became strictly parallel and never
approached, so that detector returned zero. Solving directly for the
point where three planes meet removes the parallelism problem and
yields a well-defined architectural node which is what a corner
mathematically *is*.

**What did not work and was kept out.** Histogram equalisation (CLAHE)
was considered for the saturated window regions and rejected: MASt3R
is trained on natural images and pre-normalisation risks shifting
inputs off-distribution; sensor-saturated pixels cannot be recovered
by contrast stretching. Real-time performance was explicitly out of
scope per the brief.

## Limitations

The reconstruction inherits limitations from monocular phone video.
Saturated windows, glass reflections, and motion blur produce
low-confidence regions that no post processing fully fixes. The
confidence layer surfaces these honestly rather than hiding them.

The Manhattan prior is justified only for the dominant structural
planes. Smaller furniture-like planes remain unregularised, which is
the correct behaviour but means the skeleton does not produce a
perfectly cuboid room outline.

The corner-node detector recovers ceiling side architectural nodes
more reliably than floor-side ones, because the floor plane in this
scene intersects fewer well-supported vertical planes. Much of the
lower wall is occluded by furniture.

Only one room was tested. Robustness to different indoor scenes (open
plan, kitchens, corridors) is not evaluated here.

## Beyond the brief: a navigation graph

The structural skeleton is also intended as an input to downstream
robotic tasks. Treating each corner node as a decision point and each
intersection line as a traversable edge yields a navigation graph in
which a humanoid can plan paths between corners, distances and angles
to walls are queryable in closed form from the plane equations, the
floor plane defines the locomotion surface, and the confidence layer
tells the agent where its world model is reliable. This extension is
implemented separately and is not part of the core challenge
submission.

## Repository layout

```
.
├── README.md                       this file
├── notebook.ipynb                  end-to-end pipeline
├── requirements.txt
├── figures/
│   ├── skeleton_final.png
│   ├── confidence_heatmap.png
│   ├── ransac_sweep.png
│   ├── reconstruction.png
│   ├── pipeline.png
│   ├── skeleton_rotation.gif
│   └── reconstruction_rotation.gif
└── outputs/
    ├── skeleton_final.ply          inspectable in MeshLab / 3dviewer.net
    └── clean.ply
```

## Acknowledgements

This work uses MASt3R (Wang et al., NAVER Labs Europe, 2024) for dense
reconstruction and Open3D for point-cloud operations. The challenge
brief is from the Humanoid intern recruitment process.
