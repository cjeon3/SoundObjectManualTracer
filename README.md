# Sound Object Contour Tracer: Development Methodology and Technical Documentation

UCI Hearing & Speech Lab

---

## Introduction

This document describes the development and implementation of the Sound Object Contour Tracer, a web-based application designed for manual digitization of participant drawings from auditory spatial perception experiments. The application was developed as an alternative to automated contour extraction methods, which encountered difficulties with inconsistent color detection across drawing styles, boundary-grid disambiguation, and variable brush stroke characteristics affecting geometric calculations.

The following sections detail the computational methods employed, the rationale for design decisions, approaches that were attempted and subsequently abandoned, and the final implementations retained in the current version.

---

## Coordinate System Specification

The coordinate system was designed to replicate the drawing interface used by participants during data collection. The canvas measures 1000 by 1000 pixels, representing a unit space ranging from negative ten to positive ten on both axes. The scale factor of 50 pixels per unit positions the origin at pixel coordinates (500, 500). A reference circle of three-unit radius (150 pixels) indicates the standardized head position representation.

Coordinate transformations between canvas pixels and unit space are computed as follows:

```javascript
function canvasToUnit(px, py) {
    return {
        x: (px - CENTER) / SCALE_FACTOR,
        y: (CENTER - py) / SCALE_FACTOR
    };
}

function unitToCanvas(ux, uy) {
    return {
        x: CENTER + ux * SCALE_FACTOR,
        y: CENTER - uy * SCALE_FACTOR
    };
}
```

The y-axis inversion accounts for the difference between canvas coordinates (origin at top-left, y increasing downward) and the standard Cartesian system (origin at center, y increasing upward).

---

## File Processing and Trial Assignment

### Initial Approaches

The first implementation attempted direct path parsing from ZIP archive entries, assuming a structure of `ParticipantName/TrialNumber/filename.png`. This approach failed because the archived data contained nested ZIP files and folder naming conventions that did not conform to the expected pattern.

A second approach employed regular expressions to identify numeric folder names and map them to trial numbers. This method partially succeeded but failed to parse folder names such as `Jiaxin L_11_Drawings 2`, where the suffix `_Drawings 2` was incorrectly incorporated into the trial detection logic.

### Final Implementation

The production version uses a multi-pattern parser with the following regular expression:

```
^(.+?)_(\d+)(?:_.*)?$
```

This pattern captures the participant name through a non-greedy match before the first underscore, extracts the folder number from digits following the first underscore, and discards any content following a second underscore.

Trial assignment proceeds by sorting folder numbers in ascending order and mapping them sequentially to trial identifiers. For example, folders numbered 3, 5, and 11 are assigned to Trial 1, Trial 2, and Trial 3 respectively. This approach accommodates the laboratory's data collection convention, where folder numbers represent session order rather than explicit trial labels.

The filename parser extracts frequency information from patterns such as `Jiaxin L_62.5Hz_100dB.png`, where the frequency value (62.5) is retained and the loudness specification (100dB) is recorded but not used for primary categorization.

---

## Centroid Calculation

### Drawing Speed Bias

When participants draw on a digital canvas, the application captures discrete coordinate points along the stroke path. The density of these points varies with drawing speed: slow drawing produces many closely spaced points, while rapid drawing produces fewer widely spaced points. A simple arithmetic mean of all recorded coordinates would therefore weight the centroid toward regions where the participant drew slowly, introducing systematic bias unrelated to shape geometry.

### Abandoned Approach: Simple Mean

The initial implementation computed the centroid as the arithmetic mean of all recorded points:

```javascript
// Abandoned due to drawing speed bias
let sumX = 0, sumY = 0;
for (const pt of shape) {
    sumX += pt.x;
    sumY += pt.y;
}
return { x: sumX / shape.length, y: sumY / shape.length };
```

This method was abandoned after analysis revealed that centroids were consistently displaced toward regions of higher point density, which corresponded to areas where participants had drawn more slowly or paused.

### Final Implementation: Uniform Path Resampling

The production algorithm eliminates drawing speed bias through uniform resampling along the path length. The procedure consists of four steps:

First, total path length is calculated as the sum of Euclidean distances between consecutive recorded points:

```javascript
let totalLength = 0;
const segmentLengths = [];

for (let i = 0; i < shape.length - 1; i++) {
    const dx = shape[i + 1].x - shape[i].x;
    const dy = shape[i + 1].y - shape[i].y;
    const len = Math.sqrt(dx * dx + dy * dy);
    segmentLengths.push(len);
    totalLength += len;
}
```

Second, the number of samples is determined as the greater of 100 or one sample per 0.1 units of path length:

```javascript
const numSamples = Math.max(100, Math.round(totalLength / 0.1));
const intervalLength = totalLength / numSamples;
```

This formula ensures adequate sampling density for both short and long paths. Short paths receive at least 100 samples; longer paths receive proportionally more samples to maintain consistent spacing.

Third, points are placed at equal intervals along the path using linear interpolation. For each target distance, the algorithm identifies the containing segment and computes the interpolated position:

```javascript
for (let i = 0; i < numSamples; i++) {
    const targetDist = i * intervalLength;
    
    while (segmentIdx < segmentLengths.length - 1 && 
           currentDist + segmentLengths[segmentIdx] < targetDist) {
        currentDist += segmentLengths[segmentIdx];
        segmentIdx++;
    }
    
    const segLen = segmentLengths[segmentIdx] || 0.0001;
    const t = (targetDist - currentDist) / segLen;
    const clampedT = Math.max(0, Math.min(1, t));
    
    const p1 = shape[segmentIdx];
    const p2 = shape[Math.min(segmentIdx + 1, shape.length - 1)];
    
    const resampledX = p1.x + clampedT * (p2.x - p1.x);
    const resampledY = p1.y + clampedT * (p2.y - p1.y);
    
    sumX += resampledX;
    sumY += resampledY;
}
```

The interpolation formula for a point at parameter t along a segment from P1 to P2 is:

```
x = P1.x + (P2.x - P1.x) * t
y = P1.y + (P2.y - P1.y) * t
```

Linear interpolation was selected over spline or curve-fitting methods because the recorded points already represent the participant's intended stroke; more complex interpolation could introduce smoothing artifacts that alter the perceived shape.

Fourth, the centroid is computed as the arithmetic mean of the uniformly resampled coordinates:

```javascript
return {
    x: sumX / numSamples,
    y: sumY / numSamples
};
```

Because each resampled point represents an equal arc length along the drawn path, every portion of the shape contributes equally to the centroid regardless of how quickly or slowly it was drawn.

---

## Area Calculation

Area is computed using the shoelace formula, which calculates the signed area of a polygon from its vertex coordinates:

```javascript
function calculateArea(shape) {
    if (!shape || shape.length < 3) return 0;
    let area = 0;
    for (let i = 0; i < shape.length; i++) {
        const j = (i + 1) % shape.length;
        area += shape[i].x * shape[j].y;
        area -= shape[j].x * shape[i].y;
    }
    return Math.abs(area) / 2;
}
```

The formula computes:

```
Area = (1/2) * |sum over i of (x_i * y_{i+1} - x_{i+1} * y_i)|
```

The absolute value ensures positive area regardless of vertex ordering (clockwise or counterclockwise). The modulo operation (`j = (i + 1) % shape.length`) closes the polygon by connecting the final vertex to the first.

---

## Average Shape Generation

### Abandoned Approaches

Two averaging methods were implemented and subsequently abandoned due to unacceptable artifacts in the output.

The first approach, simple point averaging, resampled all shapes to a common length and computed the arithmetic mean of corresponding points:

```javascript
// Abandoned: produces tangled contours when shapes differ in position
const maxLen = Math.max(...shapes.map(s => s.length));
const normalized = shapes.map(s => resample(s, maxLen));
const avg = [];
for (let i = 0; i < maxLen; i++) {
    let sx = 0, sy = 0;
    for (const s of normalized) {
        sx += s[i].x;
        sy += s[i].y;
    }
    avg.push({ x: sx / normalized.length, y: sy / normalized.length });
}
```

This method failed for cross-participant averaging because shapes drawn in different spatial positions produced tangled, self-intersecting average contours. Points at the same index in different shapes represented different spatial locations, and their arithmetic mean fell at positions not representative of any individual shape.

The second approach, pure centroid alignment, translated all shapes so their centroids coincided at the origin before averaging:

```javascript
// Abandoned: loses positional information
const centered = shapes.map(shape => {
    const c = calculateCentroid(shape);
    return shape.map(pt => ({ x: pt.x - c.x, y: pt.y - c.y }));
});
```

While this method eliminated the tangling artifact, it discarded information about where participants had positioned their shapes on the grid, which constituted meaningful experimental data.

### Final Implementation: Centroid-Aligned Averaging with Position Preservation

The production algorithm combines centroid alignment during the averaging process with restoration of the mean centroid position in the output. The procedure consists of four steps:

First, the centroid of each shape is calculated using the uniform path resampling method described above:

```javascript
const centroids = validShapes.map(shape => calculateCentroid(shape));

let avgCentroidX = 0, avgCentroidY = 0;
for (const c of centroids) {
    avgCentroidX += c.x;
    avgCentroidY += c.y;
}
avgCentroidX /= centroids.length;
avgCentroidY /= centroids.length;
```

Second, each shape is translated so its centroid lies at the origin, then resampled to a common length:

```javascript
const centeredResampledShapes = validShapes.map((shape, idx) => {
    const centroid = centroids[idx];
    
    const centered = shape.map(pt => ({
        x: pt.x - centroid.x,
        y: pt.y - centroid.y
    }));
    
    const resampled = [];
    for (let i = 0; i < targetLength; i++) {
        const t = i / targetLength * centered.length;
        const idx = Math.floor(t);
        const frac = t - idx;
        const p1 = centered[idx % centered.length];
        const p2 = centered[(idx + 1) % centered.length];
        resampled.push({
            x: p1.x + frac * (p2.x - p1.x),
            y: p1.y + frac * (p2.y - p1.y)
        });
    }
    return resampled;
});
```

Third, the centered shapes are averaged point by point:

```javascript
const avgCenteredShape = [];
for (let i = 0; i < targetLength; i++) {
    let sumX = 0, sumY = 0;
    for (const shape of centeredResampledShapes) {
        sumX += shape[i].x;
        sumY += shape[i].y;
    }
    avgCenteredShape.push({
        x: sumX / centeredResampledShapes.length,
        y: sumY / centeredResampledShapes.length
    });
}
```

Fourth, the average shape is translated to the mean centroid position:

```javascript
const avgShape = avgCenteredShape.map(pt => ({
    x: pt.x + avgCentroidX,
    y: pt.y + avgCentroidY
}));
```

This approach produces a clean average contour that represents the typical shape drawn by participants while preserving the average spatial position on the grid.

---

## Ground Truth Integration and Area Scaling

When connected to the original experimental database, the application can scale traced average shapes to match ground truth area measurements recorded during data collection.

The scaling transformation preserves the shape centroid while adjusting the overall size. Because area scales with the square of linear dimensions, the scale factor is computed as:

```javascript
const scaleFactor = Math.sqrt(targetArea / currentArea);
```

Each point is then scaled relative to the centroid:

```javascript
function scaleShapeToArea(shape, targetArea) {
    const currentArea = calculateArea(shape);
    if (currentArea <= 0) return shape;
    
    const scaleFactor = Math.sqrt(targetArea / currentArea);
    const centroid = calculateCentroid(shape);
    
    return shape.map(pt => ({
        x: centroid.x + (pt.x - centroid.x) * scaleFactor,
        y: centroid.y + (pt.y - centroid.y) * scaleFactor
    }));
}
```

This transformation ensures that the scaled shape maintains its geometric center and proportions while matching the recorded area from the original participant data.

---

## Cloud Storage Implementation

### Data Volume Constraints

The initial cloud storage implementation stored the complete dataset as a single JSON string in one Google Sheets cell. This approach failed when payload sizes exceeded 50,000 characters, the maximum cell capacity in Google Sheets. A typical dataset containing 14 participants with three trials each produced payloads of approximately 684,000 characters, resulting in data truncation and loss of participant records.

### Chunked Storage

The production implementation splits large JSON strings across multiple cells:

```javascript
const MAX_CELL_SIZE = 49000;
const allDataJson = JSON.stringify(allData);
const chunks = [];

for (let i = 0; i < allDataJson.length; i += MAX_CELL_SIZE) {
    chunks.push(allDataJson.substring(i, i + MAX_CELL_SIZE));
}
```

Each chunk is stored in a separate column (columns 8, 9, 10, and so on). Upon retrieval, the chunks are concatenated and parsed:

```javascript
let allDataJson = '';
for (let col = 8; col <= lastCol; col++) {
    const chunk = row[col - 1];
    if (chunk) allDataJson += chunk;
}
const allData = JSON.parse(allDataJson);
```

### Concurrency Control

To prevent data loss when multiple researchers save simultaneously, the storage operations employ script-level locking:

```javascript
const lock = LockService.getScriptLock();
try {
    lock.waitLock(30000);
    // perform save operation
} finally {
    lock.releaseLock();
}
```

The coordinate data sheet operates on an append-only basis. New records are added only if they do not duplicate existing entries, determined by comparing a signature key constructed from all data columns:

```javascript
const existingKeys = new Set();
for (const row of existingData) {
    const key = row.join('|');
    existingKeys.add(key);
}

const newRows = [];
for (const row of incomingData) {
    const key = row.join('|');
    if (!existingKeys.has(key)) {
        newRows.push(row);
    }
}
```

This design ensures that one researcher's save operation cannot overwrite another researcher's data, and repeated saves of identical data do not create duplicate records.

---

## Color Detection for Overlap Statistics

The application computes the percentage of traced contour points that overlap with colored pixels in the original participant drawing. Color classification distinguishes between canvas elements (grid lines, background) and drawn shapes.

Canvas and grid elements are identified when red, green, and blue channel values are approximately equal (within 30 units), indicating grayscale:

```javascript
const isGray = Math.abs(r - g) < 30 && Math.abs(g - b) < 30 && Math.abs(r - b) < 30;
```

Red detection encompasses pure red, pink (red with elevated brightness), and red overlaid on gray or black backgrounds:

```javascript
const isRed = (r > 150 && r > g + 50 && r > b + 50) ||
              (r > 200 && g > 150 && b > 150 && r > g && r > b) ||
              (r > g + 30 && r > b + 30 && r > 80);
```

Blue detection follows an analogous pattern for blue-dominant pixels:

```javascript
const isBlue = (b > 150 && b > r + 50 && b > g + 30) ||
               (b > 200 && r > 150 && g > 150 && b > r && b > g) ||
               (b > r + 30 && b > g + 20 && b > 80);
```

Purple pixels, where both red and blue channels are elevated while green remains low, are classified as belonging to both color categories:

```javascript
const isPurple = r > 100 && b > 100 && r > g + 20 && b > g + 20;
```

---

## Abandoned Features

Several features were implemented during development and subsequently removed from the production version.

Separate export sections for Google Sheets and Google Drive were consolidated into a unified cloud collaboration interface. The `genImg()` function, which rendered canvas contents for Drive export, was removed when the Drive export feature was deprecated. Local storage for progress persistence was replaced entirely by cloud storage to enable multi-researcher collaboration. The "Generate Current" button for single-frequency average generation was removed in favor of bulk generation with frequency selection.

---

## Data Structures

The primary data structure organizes traced shapes hierarchically by participant, trial, and frequency:

```javascript
allData = {
    "Participant Name": {
        "Trial 1": {
            "62.5Hz_100dB": {
                red: [[{x, y}, ...], ...],
                blue: [[{x, y}, ...], ...],
                redAvg: [{x, y}, ...],
                blueAvg: [{x, y}, ...]
            }
        }
    }
}
```

Cross-participant frequency averages are stored in a parallel structure indexed by frequency:

```javascript
frequencyAverages = {
    62.5: {
        redShapes: [[{x, y}, ...], ...],
        blueShapes: [[{x, y}, ...], ...],
        redAvg: [{x, y}, ...],
        blueAvg: [{x, y}, ...],
        redCentroid: {x, y},
        blueCentroid: {x, y},
        redArea: number,
        blueArea: number
    }
}
```

All coordinates are stored in unit space (negative ten to positive ten on both axes) rather than pixel coordinates, ensuring independence from display resolution.

---

## Version History

Development proceeded through four major iterations. Version 1.0 established the core tracing interface with local storage persistence. Version 2.0 introduced cloud collaboration through Google Sheets integration. Version 3.0 resolved file parsing issues for nested ZIP archives and implemented JSON chunking for large datasets. Version 4.0 added centroid-aligned averaging, the composite image gallery, and ground truth integration. Version 4.1, the current release, implemented uniform path resampling for centroid calculation and refined the color detection algorithms.

---

## References

Coordinate resampling methodology follows established practices in computational geometry for eliminating sampling bias in path-based measurements. The shoelace formula for area calculation is a standard result in polygon geometry. Linear interpolation for point placement along path segments ensures that resampled points lie exactly on the original drawn trajectory without introducing smoothing artifacts.
