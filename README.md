# Sound Object Contour Tracer: Technical Documentation

UCI Hearing & Speech Lab

Version 5.0 | February 2, 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Processing Pipeline](#processing-pipeline)
3. [Coordinate System](#coordinate-system)
4. [Data Input](#data-input)
5. [Manual Tracing](#manual-tracing)
6. [Contour Normalization](#contour-normalization)
7. [Cross-Participant Shape Averaging](#cross-participant-shape-averaging)
8. [Centroid Calculation](#centroid-calculation)
9. [Area Calculation](#area-calculation)
10. [Quality Assessment](#quality-assessment)
11. [Data Export and Persistence](#data-export-and-persistence)
12. [Verification Test Suite](#verification-test-suite)
13. [Application Settings](#application-settings)
14. [Abandoned Approaches](#abandoned-approaches)
15. [Version History](#version-history)

---

## 1. Overview

The Sound Object Contour Tracer is a browser-based application for manual digitization of participant drawings from auditory spatial perception experiments. Participants in the Sound Object Phenomenon study draw shapes representing how they perceive pure-tone stimuli presented under two interaural phase conditions: in-phase (red) and out-of-phase (blue). The application enables researchers to trace these drawings, compute cross-participant average contours at each stimulus frequency, and extract geometric measurements (area, centroid) from the averaged shapes.

The tool was developed as an alternative to automated contour extraction from PNG images, which encountered difficulties with inconsistent color detection across drawing styles, boundary-grid disambiguation, and variable brush stroke characteristics.

---

## 2. Processing Pipeline

The application implements a six-stage pipeline:

```
INPUT         TRACE         NORMALIZE       AVERAGE         MEASURE         OUTPUT
  |             |              |               |               |              |
ZIP/PNG  ->  Manual     ->  Reorder &   ->  Radial      ->  Shoelace   ->  CSV/JSON/
upload       drawing        enforce         polar            area &         PNG/Cloud
             on canvas      CCW from        averaging        resampled
                            highest Y       at mean          centroid
                                            centroid
```

Stage 1, Data Input: ZIP archives (including nested ZIPs) or individual PNG files are loaded. Folder structure encodes participant identity and trial order.

Stage 2, Manual Tracing: The researcher traces red and blue contours on a canvas overlaid on the participant's original drawing. Points are captured from mouse or stylus input and resampled to uniform density along the path.

Stage 3, Contour Normalization: Each traced contour is reordered so that index 0 corresponds to the highest-Y point (with highest-X as tiebreaker), and the winding direction is enforced as counterclockwise. This guarantees a canonical representation regardless of how the researcher drew the trace.

Stage 4, Cross-Participant Averaging: All traced shapes for a given frequency and color are averaged using radial (polar) coordinate averaging. Each shape is converted to a radial distance function relative to its centroid, these distance functions are averaged angle by angle, and the result is reconstructed at the mean centroid position.

Stage 5, Geometric Measurement: Area and centroid are computed on the average shape only. Area uses the shoelace formula. Centroid uses uniform path resampling followed by arithmetic mean.

Stage 6, Output: Results are exported as CSV (all coordinates), JSON (full state including settings), PNG composites (per-frequency visualizations), or persisted to Google Sheets for multi-researcher collaboration.

---

## 3. Coordinate System

The coordinate system replicates the drawing interface used by participants during data collection. The canvas is 1000 by 1000 pixels, representing a grid from -10 to +10 on both axes. The scale factor is 50 pixels per grid unit, with the origin at pixel (500, 500). A reference circle of 3-unit radius (150 pixels) marks the standardized head position.

Coordinate transforms between pixel space and grid space:

```
Grid to Pixel:
    pixel_x = grid_x * 50 + 500
    pixel_y = 500 - grid_y * 50

Pixel to Grid:
    grid_x = (pixel_x - 500) / 50
    grid_y = (500 - pixel_y) / 50
```

Note that the Y axis is inverted between the two systems. In grid coordinates, positive Y is upward. In pixel coordinates, positive Y is downward. The transforms account for this.

---

## 4. Data Input

### ZIP Processing

The application accepts ZIP archives containing PNG images of participant drawings. Nested ZIP files are recursively extracted. The folder structure encodes metadata:

```
ParticipantName_FolderNumber_OptionalLabel/ParticipantName_FrequencyHz_LoudnessdB.png
```

For example, `Jiaxin L_11_Drawings 2/Jiaxin L_62.5Hz_100dB.png` is parsed as participant "Jiaxin L", folder number 11, frequency 62.5 Hz.

Folder numbers determine trial order. The application sorts all folder numbers for a given participant in ascending order and maps them to sequential trial indices (Trial 1, Trial 2, ...). This allows non-contiguous folder numbering while preserving chronological order.

### Frequency Constants

The application uses six stimulus frequencies:

| Frequency (Hz) | Loudness (dB SPL) |
|---|---|
| 62.5 | 100 |
| 125 | 90 |
| 250 | 85 |
| 500 | 80 |
| 1000 | 80 |
| 2000 | 80 |

### Single File and JSON Import

Individual PNG files can be loaded directly. Previously exported JSON files can be imported to restore a complete session state, including all traced contours, settings, and metadata.

---

## 5. Manual Tracing

### Point Capture

The researcher selects a color (red or blue) and draws on the canvas using mouse or stylus input. Points are captured on every `mousemove` or `touchmove` event during an active stroke. The application supports touch input for tablet use.

### Path Resampling During Tracing

Immediately after a stroke is completed, the raw captured points are resampled to a configurable number of points (default: 1000) using arc-length parameterization. This step serves two purposes.

First, it normalizes point density along the path. Raw input points are clustered in regions where the researcher drew slowly and sparse where the researcher drew quickly. Resampling at equal arc-length intervals removes this drawing-speed bias so that every portion of the traced contour is represented by an equal number of points.

Second, it reduces storage requirements. A researcher drawing quickly might produce a few hundred points; drawing slowly might produce thousands. Resampling to a fixed count yields consistent data sizes.

The resampling algorithm works as follows:

1. Compute the total path length as the sum of Euclidean distances between consecutive points.
2. Divide the total length by (N - 1) to obtain the interval spacing, where N is the target number of points.
3. Walk along the path at uniform interval steps. For each target distance, find the segment containing that distance and use linear interpolation to compute the exact position.

Linear interpolation was chosen over spline or curve-fitting methods because the raw captured points already represent the researcher's intended stroke. Higher-order interpolation could introduce smoothing artifacts that alter the traced shape.

---

## 6. Contour Normalization

Every contour, whether an individual trace or an averaged result, is normalized to a canonical form before use. Normalization ensures that contours traced by different researchers, or the same researcher on different occasions, are directly comparable regardless of where the stroke began or which direction it was drawn.

### Starting Point

The contour is rotated (cyclically reindexed) so that index 0 corresponds to the point with the highest Y coordinate in grid space. If multiple points share the highest Y, the one with the highest X coordinate is chosen. This places the starting point at the topmost location on the shape, with rightward preference as the tiebreaker.

### Winding Direction

After rotating to the correct starting point, the winding direction is checked using the signed area formula:

```
SignedArea = (1/2) * sum_i (x_i * y_{i+1} - x_{i+1} * y_i)
```

A positive signed area indicates counterclockwise traversal. A negative signed area indicates clockwise traversal. If the contour is clockwise, the point order is reversed (keeping the starting point at index 0) to produce counterclockwise winding.

### Why This Matters

Without normalization, two identical shapes traced in opposite directions would cancel each other during point-by-point averaging. Normalization prevents this by ensuring all shapes share the same traversal direction and reference point before any averaging takes place. The radial averaging method (Section 7) is inherently direction-agnostic, so normalization is primarily relevant for data export consistency and display.

---

## 7. Cross-Participant Shape Averaging

### Method: Radial (Polar) Coordinate Averaging

The application averages shapes by converting them to polar coordinate representations relative to their centroids, averaging the radial distance functions, and reconstructing the result. This approach avoids the point correspondence problem entirely.

### Intuition

Consider two shapes drawn by different participants. One is roughly circular, the other roughly elliptical. Both are supposed to represent the same percept at the same frequency. The question is: what does the "average" shape look like?

A naive approach would be to match points along the two contours and average their positions. But which point on the circle corresponds to which point on the ellipse? The two shapes may have been traced starting at different locations, in different directions, and with different numbers of raw points. Any attempt to match points by index is arbitrary and produces artifacts, especially self-intersecting loops.

The radial method sidesteps this entirely. Instead of asking "which points correspond," it asks "how far does each shape extend in each direction from its center?" This is a well-defined quantity for any closed shape. By averaging these directional extents, the result captures the typical shape without requiring any point matching.

### Algorithm

The averaging proceeds in seven steps.

Step 1, Centroid Calculation: The centroid of each input shape is computed using uniform path resampling (see Section 8).

Step 2, Radial Conversion: For each shape, a ray is cast from the centroid at each of N uniformly spaced angles (where N is the radial resolution, default 720). The angles span the full circle: angle_k = (k / N) * 2 * pi, for k = 0, 1, ..., N-1. For each ray, the algorithm finds all intersections with the shape boundary segments using ray-segment intersection testing, and records the distance to the farthest intersection. The result is a function d_shape(angle) mapping each angle to a radial distance.

Step 3, Mean Centroid: The average centroid position is computed as the arithmetic mean of all input shape centroids. This preserves the average spatial position of the original drawings, which is meaningful experimental data (it reflects where participants perceived the sound object to be).

Step 4, Distance Averaging: At each angle, the radial distances from all input shapes are averaged:

```
d_avg(angle_k) = (1 / M) * sum_{j=1}^{M} d_j(angle_k)
```

where M is the number of input shapes and d_j(angle_k) is the radial distance for shape j at angle k.

Step 5, Cartesian Reconstruction: The averaged radial distances are converted back to Cartesian coordinates, positioned at the mean centroid:

```
x_k = centroid_x_avg + d_avg(angle_k) * cos(angle_k)
y_k = centroid_y_avg + d_avg(angle_k) * sin(angle_k)
```

Step 6, Normalization: The reconstructed contour is normalized (Section 6) to start at the highest Y point and traverse counterclockwise.

Step 7, Verification: A defensive check confirms that the signed area of the output is positive (counterclockwise). If normalization failed for any reason, the shape is reversed. In practice this check has never been triggered, but it guards against numerical edge cases.

### Ray-Segment Intersection

Each ray-boundary intersection is computed using a standard parametric formulation. Given a ray origin (r_x, r_y) with direction (d_x, d_y), and a segment from (p1_x, p1_y) to (p2_x, p2_y), the intersection parameters t (distance along ray) and u (fraction along segment) are:

```
denom = d_x * (p2_y - p1_y) - d_y * (p2_x - p1_x)

t = ((p1_x - r_x) * (p2_y - p1_y) - (p1_y - r_y) * (p2_x - p1_x)) / denom
u = ((p1_x - r_x) * d_y - (p1_y - r_y) * d_x) / denom
```

The intersection is valid when t >= 0 (the intersection is in the ray's forward direction) and 0 <= u <= 1 (the intersection lies on the segment). When |denom| < 1e-10, the ray and segment are treated as parallel and no intersection is recorded.

For each angle, the farthest valid intersection (largest t) is used. Taking the farthest rather than the nearest ensures that the radial sample captures the outer boundary of the shape, not an inner fold or crossing.

### Radial Resolution

The number of angular samples N is configurable via a slider in the interface, with range 36 to 3600 and default 720. Higher resolution yields more precise shape reconstruction at the cost of computation time. At the default of 720, the angular spacing is 0.5 degrees.

The angle conversion uses the formula angle_rad = (k / N) * 2 * pi to ensure the full circle is always covered exactly once, regardless of the value of N.

### Why Radial Averaging Over Alternatives

Radial averaging was chosen after evaluating and discarding two other approaches (see Section 14). Its key properties are:

1. It requires no point correspondence. Each shape is represented by an independent function of angle.
2. It is inherently direction-agnostic. A shape drawn clockwise defines the same boundary as one drawn counterclockwise; raycasting finds the same edges either way.
3. It cannot produce self-intersecting results. The output is a single-valued function of angle relative to the centroid, which by definition traces a simple closed curve.
4. It preserves average position. The mean centroid is used for reconstruction, so the output reflects where participants collectively placed their drawings on the grid.
5. It preserves average size. If all inputs have area A, the output area is approximately A (up to discretization effects from angular sampling).

The primary limitation of radial averaging is its assumption that the shape is star-convex with respect to its centroid: every ray from the centroid must intersect the boundary. For highly concave or multi-lobed shapes, some rays may miss the boundary entirely (yielding distance 0 at that angle). In practice, the hand-drawn shapes in this study are sufficiently convex that this limitation does not arise.

NOTE: The radial averaging function (simpleAverageShapes) does not use the traced points directly as its output. It casts rays from each shape's centroid at radialResolution uniformly spaced angles (default 720), finds the intersection distances, averages those distances, and reconstructs the result as 720 new Cartesian points. The original traced point count and spacing are irrelevant to the output.

So if a contour was traced with 1000 resampled points, the averaging pipeline produces a 720-point (or whatever the slider is set to) radial reconstruction. The manual trace is only used as the boundary that the rays intersect against.

---

## 8. Centroid Calculation

The centroid of a contour is calculated using uniform path resampling followed by arithmetic mean. This method was chosen to eliminate drawing-speed bias.

### The Problem with Raw Point Averaging

When a researcher traces a contour, the captured points are not uniformly distributed along the path. Regions drawn slowly produce many closely spaced points; regions drawn quickly produce few widely spaced points. If the centroid were computed as the simple mean of these raw points, slow-drawn regions would contribute disproportionately, pulling the centroid toward wherever the researcher happened to pause or slow down. This is an artifact of the input device, not a property of the shape.

### The Solution: Uniform Path Resampling

The algorithm resamples the contour at equal arc-length intervals before computing the mean, so that every portion of the path contributes equally regardless of original point density.

Step 1: The total path length L is computed as the sum of Euclidean distances between consecutive points:

```
L = sum_{i=0}^{n-2} sqrt((x_{i+1} - x_i)^2 + (y_{i+1} - y_i)^2)
```

Step 2: The number of resample points is set to max(100, L / 0.1). This gives at least 100 samples for short paths and approximately one sample per 0.1 grid units for longer paths.

Step 3: Resample points are placed at equal intervals along the path using linear interpolation. For each target distance d along the path, the algorithm finds the segment containing d and interpolates:

```
x = x_a + t * (x_b - x_a)
y = y_a + t * (y_b - y_a)
```

where (x_a, y_a) and (x_b, y_b) are the segment endpoints and t is the fractional distance within that segment.

Step 4: The centroid is the arithmetic mean of the resampled coordinates:

```
centroid_x = (1/N) * sum_{i=1}^{N} x_i
centroid_y = (1/N) * sum_{i=1}^{N} y_i
```

Because each resampled point represents an equal length of the path, every section of the contour contributes equally to the centroid.

---

## 9. Area Calculation

Area is computed using the shoelace formula (also known as Gauss's area formula or the surveyor's formula). Given a polygon defined by vertices (x_0, y_0), (x_1, y_1), ..., (x_{n-1}, y_{n-1}), the area is:

```
Area = (1/2) * |sum_{i=0}^{n-1} (x_i * y_{i+1} - x_{i+1} * y_i)|
```

where index arithmetic is modulo n so that the last vertex connects back to the first.

### Intuition

The formula computes the area by summing the signed areas of trapezoids formed between each edge of the polygon and the x-axis. Moving left to right along the top of the polygon adds positive area; moving right to left along the bottom subtracts, leaving exactly the enclosed area. The crisscross pattern of the multiplications (x_i with y_{i+1}, then x_{i+1} with y_i) resembles the lacing of a shoe, giving the formula its name. The absolute value ensures a positive result regardless of traversal direction.

### Why Shoelace Rather Than Brush-Corrected Area

Earlier versions of the application included a brush-width correction that added (perimeter * brush_radius) to the shoelace area, approximating the visual area of the original brush strokes on screen. This was removed in v5.0 for two reasons.

First, the averaged contours produced by radial averaging are mathematical boundaries, not rendered brush strokes. They represent the geometric boundary of the average shape, and the shoelace formula is the mathematically appropriate measure for the area enclosed by a polygon.

Second, the brush-correction introduced a dependency on brush size that was not meaningful for the averaged result. The brush size is a display parameter of the tracing tool, not a property of the participant's perceived sound object. Removing the correction yields area values that depend only on the shape geometry.

A 2x2 grid-unit square now reports area 4.0 (the geometric area of the polygon), without any additive correction term.

### Where Area Is Calculated

Area is computed on the average shape only, not on individual traced contours. Individual traces are intermediate data; the scientifically relevant quantity is the area of the cross-participant average at each frequency and phase condition.

---

## 10. Quality Assessment

### Image Overlap Statistics

After tracing, the application computes what percentage of traced points fall on colored pixels in the original participant drawing. This serves as a quality check on tracing accuracy.

For each traced point, the algorithm converts to canvas pixel coordinates and checks a neighborhood (sized by the brush radius) for the presence of the target color in the original image. The overlap percentage is reported per color.

### Color Detection

The original participant drawings use red for in-phase and blue for out-of-phase contours. However, PNG compression, anti-aliasing, and drawing tool variations produce a wide range of pixel colors. The detection functions use multi-condition thresholding to handle these variations:

Red detection covers pure red, pink, salmon, magenta (counted as half red/half blue), and other reddish hues while excluding brown and near-gray tones.

Blue detection covers pure blue, cyan, steel blue, periwinkle, and other bluish hues while excluding near-gray tones.

Purple pixels (roughly equal red and blue components) are counted as both red and blue, since they typically arise at overlapping boundaries.

---

## 11. Data Export and Persistence

### CSV Export

Exports all traced coordinate data in a flat table:

```
Participant, Trial, Frequency, Color, Shape, Point, X, Y, IsAvg
```

Individual shapes are numbered sequentially. Average shapes are labeled "Avg" in the Shape column and flagged with IsAvg=true. Coordinates are in grid units with six decimal places.

### JSON Export

Exports the complete application state including all traced data, settings (resample rate, image opacity, shape opacity, brush size, radial resolution), and metadata (export date, version number). This file can be re-imported to restore a full session.

### PNG Export

Composite images can be exported per frequency. Each composite shows all individual traced contours at reduced opacity with the average shape drawn prominently on top. A grid and reference circle are included for spatial reference. Area and centroid statistics are overlaid as text.

### Google Sheets Cloud Collaboration

The application integrates with a Google Apps Script web application for multi-researcher collaboration. The cloud backend provides:

1. Save: Coordinate data for all traced shapes is serialized to JSON and sent to the Apps Script endpoint. Large datasets are chunked to stay within URL length limits. Data is stored in a Google Sheets spreadsheet with separate sheets for coordinate data and summary statistics.

2. Load: Previously saved data can be retrieved by specifying the tracer name. The application merges loaded data with any existing local data, avoiding duplicates.

3. Auto-save: An optional auto-save mode triggers a cloud save after each completed stroke, with a debounce delay to avoid excessive network requests.

Each researcher identifies themselves by a tracer name, and the cloud backend stores data keyed by this name. This allows multiple researchers to contribute traces to a shared dataset.

---

## 12. Verification Test Suite

The application includes four automated tests (accessible via the browser console as `runContourTests()`) that verify the correctness of the normalization and averaging pipeline.

### Test 1: Normalization of CW and CCW Input

Two representations of the same 2x2 square at position (-5, 5) to (-3, 3) are provided, one drawn clockwise and one counterclockwise. Both are normalized and checked to confirm:

- Starting point is (-3, 5), the highest-Y point with highest-X tiebreaker.
- Signed area is positive (counterclockwise).
- Both normalized outputs are identical point sequences.

### Test 2: Averaging Shapes at Different Positions

Two identical 2x2 squares are placed at opposite corners of the grid: one at (-6, 6) to (-4, 4) drawn clockwise, one at (4, -4) to (6, -6) drawn counterclockwise. The average is checked for:

- Centroid near (0, 0), the midpoint of the two input centroids.
- Area near 4.0 square units (preserved from input).
- Starting point at the highest Y, with counterclockwise winding.

### Test 3: Non-Highest-Y Start with Clockwise Input

Two identical 2x2 squares centered at the origin are drawn starting from non-topmost vertices and in clockwise direction. This is a worst-case scenario for normalization. The test verifies that both normalize identically (starting at (1, 1), counterclockwise), and that their average has centroid near (0, 0), area near 4.0, and correct normalization.

### Test 4: Opposite Positions with Opposite Winding Must Not Cancel

A 2x2 square centered at (1, 1) is drawn clockwise. A 2x2 square centered at (-1, -1) is drawn counterclockwise. This tests whether opposite winding directions cause destructive cancellation in the averaging. The critical check is that the averaged area is greater than 2.0 (it should be near 4.0). This test confirms that the radial averaging method, which operates on distances rather than signed displacements, is immune to direction-dependent cancellation.

---

## 13. Application Settings

| Setting | Default | Range | Description |
|---|---|---|---|
| Resample Rate | 1000 | 500 - 10000 | Number of points per traced contour after resampling |
| Brush Size | 5 px | 1 - 20 | Drawing brush diameter in pixels |
| Image Opacity | 50% | 0 - 100% | Transparency of the background participant drawing |
| Shape Opacity | 30% | 0 - 100% | Transparency of previously traced contours |
| Radial Resolution | 720 | 36 - 3600 | Number of angular samples for radial averaging |

Radial resolution controls the angular precision of the averaging algorithm. At 720 (default), angles are spaced 0.5 degrees apart. Lower values (e.g. 360 for 1-degree spacing) are faster but less precise. Higher values (e.g. 3600 for 0.1-degree spacing) give finer shape reconstruction. For the shapes in this study, 720 provides a good balance.

All settings are included in JSON exports and cloud saves, allowing sessions to be restored with the same configuration.

---

## 14. Abandoned Approaches

Three averaging methods were implemented and subsequently replaced due to unacceptable artifacts.

### Simple Point Averaging (v1.x)

All shapes were resampled to a common point count and averaged index by index. This failed for cross-participant averaging because shapes at different spatial positions produced tangled, self-intersecting contours. Points at the same index in different shapes represented different angular positions around the shape, and their mean fell at positions not representative of any individual shape.

### Centroid-Aligned Point Averaging (v3.x)

Shapes were translated so their centroids coincided at the origin, then resampled and averaged point by point. This eliminated the tangling artifact but introduced a subtler problem: shapes drawn in different directions or starting at different locations still produced self-intersecting loops. Point indices still lacked angular correspondence, so a point near the top of one shape could be averaged with a point near the bottom of another.

An angular alignment step was added (rotating point indices so index 0 corresponded to the rightmost point, then enforcing counterclockwise winding), which improved results but did not fully eliminate self-intersections in edge cases with highly dissimilar shapes.

### Brush-Width Area Correction (v4.3 - v4.9)

Area was calculated as shoelace area plus (perimeter * brush_radius), approximating the visual area of the brush stroke. This was appropriate when measuring individual drawn shapes rendered on screen, but it was not appropriate for averaged contours, which are mathematical constructions with no brush rendering. The correction was removed in v5.0 so that area values reflect pure polygon geometry.
