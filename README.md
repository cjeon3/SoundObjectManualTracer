# Sound Object Contour Tracer: Development Methodology and Technical Documentation

---

## Processing Pipeline Overview

The Sound Object Contour Tracer implements a multi-stage pipeline for digitizing and analyzing participant drawings from auditory spatial perception experiments:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA INPUT STAGE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  ZIP Upload → Nested ZIP Extraction → Folder Pattern Parsing                │
│      ↓              ↓                       ↓                               │
│  PNG Files    Trial Assignment        Participant ID                        │
│      └──────────────┴───────────────────────┘                               │
│                      ↓                                                      │
│              Image Loading & Tab Generation                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MANUAL TRACING STAGE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  Canvas Drawing → Point Capture → Coordinate Transform → Path Resampling   │
│       ↓               ↓                  ↓                    ↓             │
│  Mouse/Touch     Raw Pixels      Canvas→Unit Space      Uniform Density    │
│  Events          (1000×1000)     (-10 to +10)           (configurable)     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                         QUALITY ASSESSMENT STAGE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  Image Pixel Analysis → Color Detection → Overlap Calculation               │
│         ↓                     ↓                   ↓                         │
│  Original Drawing      Red/Blue/Purple      Percentage of traced           │
│  RGB Values            Classification       points on colored pixels       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CROSS-PARTICIPANT AVERAGING STAGE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  Shape Collection → Centroid Calculation → Centroid Alignment → Averaging  │
│        ↓                    ↓                     ↓                ↓        │
│  All shapes at       Uniform path            Translate to       Point-by-  │
│  frequency X         resampling              origin             point mean │
│                            ↓                                        ↓      │
│                   Eliminates drawing                          Translate to │
│                   speed bias                                  mean centroid│
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                      GROUND TRUTH SCALING STAGE (Optional)                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  Original Data Fetch → Area Extraction → Scale Factor → Shape Scaling      │
│         ↓                    ↓                ↓               ↓             │
│  Google Sheets API    Average area per   √(target/current)  Preserve       │
│  Connection           frequency/color                       centroid       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                           OUTPUT STAGE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  Composite Gallery → PNG Export → CSV/JSON Export → Cloud Persistence      │
│        ↓                 ↓              ↓                   ↓               │
│  Visual preview     Per-frequency   All coordinates    Google Sheets       │
│  with statistics    images          with metadata      collaboration       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Introduction

This document describes the development and implementation of the Sound Object Contour Tracer, a web-based application designed for manual digitization of participant drawings from auditory spatial perception experiments. The application was developed as an alternative to automated contour extraction methods, which encountered difficulties with inconsistent color detection across drawing styles, boundary-grid disambiguation, and variable brush stroke characteristics affecting geometric calculations.

The following sections detail the computational methods employed, the rationale for design decisions, approaches that were attempted and subsequently abandoned, and the final implementations retained in the current version.

---

## Coordinate System Specification

The coordinate system replicates the drawing interface used by participants during data collection. The canvas measures 1000 by 1000 pixels, representing a unit space ranging from negative ten to positive ten on both axes. The scale factor of 50 pixels per unit positions the origin at pixel coordinates (500, 500). A reference circle of three-unit radius (150 pixels) indicates the standardized head position representation.

Coordinate transformations between canvas pixels and unit space are computed as follows:

```javascript
const CANVAS_SIZE = 1000;
const UNIT_RANGE = 10;
const SCALE_FACTOR = CANVAS_SIZE / (UNIT_RANGE * 2); // = 50
const CENTER = CANVAS_SIZE / 2; // = 500

function canvasToUnit(x, y) {
    return {
        x: (x - CENTER) / SCALE_FACTOR,
        y: (CENTER - y) / SCALE_FACTOR
    };
}

function unitToCanvas(x, y) {
    return {
        x: x * SCALE_FACTOR + CENTER,
        y: CENTER - y * SCALE_FACTOR
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

Supported frequencies are defined as:

```javascript
const FREQUENCIES = [
    { hz: 62.5, db: 100 },
    { hz: 125, db: 90 },
    { hz: 250, db: 85 },
    { hz: 500, db: 80 },
    { hz: 1000, db: 80 },
    { hz: 2000, db: 80 }
];
```

---

## Path Resampling

When participants draw on a digital canvas, the application captures discrete coordinate points along the stroke path. The density of these points varies with drawing speed: slow drawing produces many closely spaced points, while rapid drawing produces fewer widely spaced points.

The resampling function normalizes point density across all traced shapes:

```javascript
function resample(pts, n) {
    if (pts.length < 2) return pts;
    
    // Calculate total path length
    let len = 0;
    const segs = [];
    for (let i = 0; i < pts.length - 1; i++) {
        const d = Math.hypot(pts[i+1].x - pts[i].x, pts[i+1].y - pts[i].y);
        segs.push(d);
        len += d;
    }
    if (len === 0) return [pts[0]];
    
    // Generate n evenly-spaced points along the path
    const res = [{ ...pts[0] }];
    const iv = len / (n - 1);
    let cd = 0, si = 0;
    
    for (let i = 1; i < n - 1; i++) {
        const td = i * iv;
        while (si < segs.length && cd + segs[si] < td) {
            cd += segs[si++];
        }
        if (si >= segs.length) break;
        const pr = (td - cd) / segs[si];
        res.push({
            x: pts[si].x + (pts[si+1].x - pts[si].x) * pr,
            y: pts[si].y + (pts[si+1].y - pts[si].y) * pr
        });
    }
    res.push({ ...pts[pts.length - 1] });
    return res;
}
```

The default resample rate is 1000 points, configurable via slider from 500 to 10000 points.

---

## Centroid Calculation

### Drawing Speed Bias Problem

A simple arithmetic mean of all recorded points would weight the centroid toward regions where the participant drew slowly, introducing systematic bias unrelated to shape geometry.

### Abandoned Approach: Simple Mean

The initial implementation computed the centroid as the arithmetic mean of all recorded points. This method was abandoned after analysis revealed that centroids were consistently displaced toward regions of higher point density.

### Final Implementation: Uniform Path Resampling

The production algorithm eliminates drawing speed bias through uniform resampling along the path length:

```javascript
function calculateCentroid(shape) {
    if (!shape || shape.length === 0) return { x: 0, y: 0 };
    if (shape.length === 1) return { x: shape[0].x, y: shape[0].y };
    
    // Step 1: Calculate total path length
    let totalLength = 0;
    const segmentLengths = [];
    
    for (let i = 0; i < shape.length - 1; i++) {
        const dx = shape[i + 1].x - shape[i].x;
        const dy = shape[i + 1].y - shape[i].y;
        const len = Math.sqrt(dx * dx + dy * dy);
        segmentLengths.push(len);
        totalLength += len;
    }
    
    // Handle degenerate case
    if (totalLength < 0.0001) {
        return { x: shape[0].x, y: shape[0].y };
    }
    
    // Step 2: Determine number of samples
    const numSamples = Math.max(100, Math.round(totalLength / 0.1));
    const intervalLength = totalLength / numSamples;
    
    // Step 3: Resample at equal intervals using linear interpolation
    let sumX = 0, sumY = 0;
    let currentDist = 0, segmentIdx = 0;
    
    for (let i = 0; i < numSamples; i++) {
        const targetDist = i * intervalLength;
        
        while (segmentIdx < segmentLengths.length - 1 && 
               currentDist + segmentLengths[segmentIdx] < targetDist) {
            currentDist += segmentLengths[segmentIdx];
            segmentIdx++;
        }
        
        const segLen = segmentLengths[segmentIdx] || 0.0001;
        const t = segLen > 0 ? (targetDist - currentDist) / segLen : 0;
        const clampedT = Math.max(0, Math.min(1, t));
        
        const p1 = shape[segmentIdx];
        const p2 = shape[Math.min(segmentIdx + 1, shape.length - 1)];
        
        sumX += p1.x + clampedT * (p2.x - p1.x);
        sumY += p1.y + clampedT * (p2.y - p1.y);
    }
    
    // Step 4: Compute arithmetic mean
    return {
        x: sumX / numSamples,
        y: sumY / numSamples
    };
}
```

The formula `numSamples = max(100, totalLength / 0.1)` ensures adequate sampling density for both short and long paths.

---

## Area Calculation

Area is computed using the shoelace formula:

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

The absolute value ensures positive area regardless of vertex ordering.

---

## Average Shape Generation

### Abandoned Approaches

**Simple Point Averaging**: Resampled all shapes to a common length and computed the arithmetic mean of corresponding points. This failed for cross-participant averaging because shapes drawn in different spatial positions produced tangled, self-intersecting average contours.

**Pure Centroid Alignment**: Translated all shapes so their centroids coincided at the origin before averaging. While this eliminated tangling, it discarded positional information that constituted meaningful experimental data.

### Final Implementation: Centroid-Aligned Averaging with Position Preservation

The production algorithm combines centroid alignment during averaging with restoration of the mean centroid position:

```javascript
function simpleAverageShapes(shapes, targetLength = 1000) {
    if (!shapes || shapes.length === 0) return null;
    
    const validShapes = shapes.filter(s => s && s.length >= 3);
    if (validShapes.length === 0) return null;
    if (validShapes.length === 1) return validShapes[0];
    
    // Step 1: Calculate centroid of each shape
    const centroids = validShapes.map(shape => calculateCentroid(shape));
    
    // Calculate average centroid position
    let avgCentroidX = 0, avgCentroidY = 0;
    for (const c of centroids) {
        avgCentroidX += c.x;
        avgCentroidY += c.y;
    }
    avgCentroidX /= centroids.length;
    avgCentroidY /= centroids.length;
    
    // Step 2: Translate shapes to origin and resample
    const centeredResampledShapes = validShapes.map((shape, idx) => {
        const centroid = centroids[idx];
        const centered = shape.map(pt => ({
            x: pt.x - centroid.x,
            y: pt.y - centroid.y
        }));
        
        // Resample to target length
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
    
    // Step 3: Average point-by-point
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
    
    // Step 4: Translate to average centroid position
    return avgCenteredShape.map(pt => ({
        x: pt.x + avgCentroidX,
        y: pt.y + avgCentroidY
    }));
}
```

This approach produces a clean average contour representing the typical shape while preserving the average spatial position.

---

## Ground Truth Integration and Area Scaling

When connected to the original experimental database via Google Apps Script API, the application can scale traced average shapes to match ground truth area measurements.

The scaling transformation preserves the shape centroid while adjusting overall size:

```javascript
function scaleShapeToArea(shape, targetArea) {
    if (!shape || shape.length < 3 || targetArea <= 0) return shape;
    
    const currentArea = calculateArea(shape);
    if (currentArea <= 0) return shape;
    
    // Area scales with square of linear scale
    const scaleFactor = Math.sqrt(targetArea / currentArea);
    const centroid = calculateCentroid(shape);
    
    return shape.map(pt => ({
        x: centroid.x + (pt.x - centroid.x) * scaleFactor,
        y: centroid.y + (pt.y - centroid.y) * scaleFactor
    }));
}
```

---

## Color Detection for Overlap Statistics

The application computes the percentage of traced contour points that overlap with colored pixels in the original participant drawing. This provides quality metrics for tracing accuracy.

### Canvas and Grid Detection

Canvas elements (grid lines, background) are identified when RGB channels are approximately equal:

```javascript
function isCanvasOrGrid(r, g, b) {
    const maxDiff = Math.max(Math.abs(r - g), Math.abs(g - b), Math.abs(r - b));
    return maxDiff < 30;
}
```

### Red Detection

Red detection encompasses pure red, pink, magenta, and red overlaid on gray or black backgrounds:

```javascript
function hasRedComponent(r, g, b) {
    if (isCanvasOrGrid(r, g, b)) return false;
    
    // Pure red or red-dominant
    if (r > 150 && r > g * 1.2 && r > b * 1.2) return true;
    
    // Pink (red + white blend)
    if (r > 180 && g > 100 && b > 100 && r > g && r > b) return true;
    
    // Magenta/pink
    if (r > 150 && b > 100 && r > g * 1.3 && g < 150) return true;
    
    // Red over gray
    const avgGB = (g + b) / 2;
    if (r > avgGB + 40 && r > 100) return true;
    
    // Red over black
    if (r > 50 && r > g * 1.5 && r > b * 1.5 && g < 100 && b < 100) return true;
    
    // Light red/salmon
    if (r > 200 && g > 120 && g < 180 && b > 100 && b < 160 && r > g && r > b) return true;
    
    return false;
}
```

### Blue Detection

Blue detection follows an analogous pattern:

```javascript
function hasBlueComponent(r, g, b) {
    if (isCanvasOrGrid(r, g, b)) return false;
    
    // Pure blue or blue-dominant
    if (b > 150 && b > r * 1.2 && b > g * 1.1) return true;
    
    // Cyan-blue
    if (b > 150 && g > 100 && b >= g && r < g) return true;
    
    // Light blue
    if (b > 180 && g > 100 && r > 80 && b > r && b > g) return true;
    
    // Blue over gray
    const avgRG = (r + g) / 2;
    if (b > avgRG + 40 && b > 100) return true;
    
    // Blue over black
    if (b > 50 && b > r * 1.5 && b > g * 1.3 && r < 100) return true;
    
    // Steel blue / muted blue
    if (b > 120 && b > r && b > g * 0.9 && r < 150) return true;
    
    return false;
}
```

### Purple Detection

Purple pixels are classified as belonging to both red and blue categories:

```javascript
function isPurple(r, g, b) {
    if (isCanvasOrGrid(r, g, b)) return false;
    
    // Purple: significant red and blue, less green
    if (r > 100 && b > 100 && r > g * 1.2 && b > g * 1.2) return true;
    
    // Magenta
    if (r > 150 && b > 150 && g < 120) return true;
    
    // Violet
    if (b > 120 && r > 80 && b > g && r > g && g < 150) return true;
    
    // Light purple/lavender
    if (r > 150 && b > 150 && g > 100 && g < r && g < b) return true;
    
    return false;
}
```

### Overlap Calculation

The overlap statistic is calculated by checking each traced point against a small area (brush radius) in the original image:

```javascript
function calculateImageOverlap(img, data) {
    // ... setup temporary canvas and get image data ...
    
    let redOverlap = null;
    if (data.red && data.red.length > 0) {
        let totalRedPoints = 0;
        let matchingRedPoints = 0;
        
        data.red.forEach(shape => {
            if (!shape) return;
            shape.forEach(pt => {
                const canvasPt = unitToCanvas(pt.x, pt.y);
                totalRedPoints++;
                
                // Check area around the point
                let found = false;
                for (let dx = -brushSize; dx <= brushSize && !found; dx++) {
                    for (let dy = -brushSize; dy <= brushSize && !found; dy++) {
                        if (isRedPixel(canvasPt.x + dx, canvasPt.y + dy)) {
                            found = true;
                        }
                    }
                }
                if (found) matchingRedPoints++;
            });
        });
        
        redOverlap = totalRedPoints > 0 ? (matchingRedPoints / totalRedPoints) * 100 : 0;
    }
    
    // Similar calculation for blue...
    
    return { red: redOverlap, blue: blueOverlap };
}
```

---

## Cloud Storage Implementation

### Data Volume Constraints

The initial implementation stored the complete dataset as a single JSON string in one Google Sheets cell. This failed when payload sizes exceeded 50,000 characters.

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

### Concurrency Control

Storage operations employ script-level locking to prevent data loss from simultaneous saves. The coordinate data sheet operates on an append-only basis with duplicate detection.

### Data Merging

When loading data from multiple tracers, the application merges without replacing existing shapes:

```javascript
function mergeDataInto(target, source) {
    for (const participant of Object.keys(source)) {
        // ... ensure structure exists ...
        
        // Merge shapes, avoiding exact duplicates
        srcData.red.forEach(srcShape => {
            const isDuplicate = tgtData.red.some(tgtShape => 
                tgtShape && tgtShape.length === srcShape.length &&
                tgtShape.every((pt, i) => pt.x === srcShape[i].x && pt.y === srcShape[i].y)
            );
            if (!isDuplicate) {
                tgtData.red.push(srcShape);
            }
        });
        // Similar for blue...
    }
}
```

---

## Visual Rendering

### Layer Order

The canvas renders in the following order (bottom to top):

1. White background
2. Background image at configurable opacity (default 50%)
3. Reference circle (3-unit radius)
4. Traced shapes at configurable opacity (default 30%)
5. Average shapes at full opacity with darker colors
6. Grid overlay

### Configurable Opacities

- **Image Opacity**: 0-100%, controls visibility of original participant drawing
- **Shape Opacity**: 0-100%, controls visibility of individual traced shapes behind the average

### Composite Image Generation

Frequency composite images display all individual traced shapes at low opacity with the calculated average shape prominently overlaid:

```javascript
function updateFreqCompositesGallery() {
    // For each frequency with data:
    // 1. Draw all individual red shapes at traceOpacity (0.25)
    // 2. Draw all individual blue shapes at traceOpacity (0.25)
    // 3. Draw red average shape at full opacity with thick line
    // 4. Draw blue average shape at full opacity with thick line
    // 5. Draw centroid markers (+ signs) for each average
}
```

---

## Data Structures

### Primary Data Structure

Traced shapes are organized hierarchically:

```javascript
allData = {
    "Participant Name": {
        "Trial 1": {
            "62.5Hz_100dB": {
                red: [[{x, y}, ...], ...],    // Array of shapes
                blue: [[{x, y}, ...], ...],   // Array of shapes
                redAvg: [{x, y}, ...],        // Per-trial average (optional)
                blueAvg: [{x, y}, ...]        // Per-trial average (optional)
            }
        }
    }
}
```

### Cross-Participant Frequency Averages

```javascript
frequencyAverages = {
    62.5: {
        redShapes: [[{x, y}, ...], ...],  // All red shapes at this frequency
        blueShapes: [[{x, y}, ...], ...], // All blue shapes at this frequency
        redAvg: [{x, y}, ...],            // Cross-participant average
        blueAvg: [{x, y}, ...],           // Cross-participant average
        redCentroid: {x, y},              // Centroid of average shape
        blueCentroid: {x, y},             // Centroid of average shape
        redArea: number,                   // Area of average shape
        blueArea: number                   // Area of average shape
    }
}
```

All coordinates are stored in unit space (-10 to +10 on both axes).

---

## Export Formats

### CSV Export

```
Participant,Trial,Frequency,Color,Shape,Point,X,Y,IsAvg
Jiaxin L,Trial 1,62.5Hz_100dB,Red,1,1,-2.340000,3.120000,false
...
Jiaxin L,Trial 1,62.5Hz_100dB,Red,Avg,1,-2.150000,3.080000,true
```

### JSON Export

```json
{
    "exportDate": "2026-01-14T...",
    "version": "4.2",
    "settings": {
        "resampleRate": 1000,
        "imageOpacity": 50,
        "shapeOpacity": 30,
        "brushSize": 5
    },
    "allData": { ... }
}
```

### PNG Export

Per-frequency composite images include:
- All individual traced shapes at configurable opacity
- Average shapes at full opacity with centroid markers
- Grid overlay
- Statistics text (shape counts, centroid coordinates, areas, centroid shift)

---

## Version History

- **Version 1.0**: Core tracing interface with local storage persistence
- **Version 2.0**: Cloud collaboration through Google Sheets integration
- **Version 3.0**: Nested ZIP support, JSON chunking for large datasets
- **Version 4.0**: Centroid-aligned averaging, composite image gallery, ground truth integration
- **Version 4.1**: Uniform path resampling for centroid calculation, refined color detection
- **Version 4.2**: Enhanced tracing statistics with overlap calculation, improved visual feedback, cloud save progress indicators

---

## References

Coordinate resampling methodology follows established practices in computational geometry for eliminating sampling bias in path-based measurements. The shoelace formula for area calculation is a standard result in polygon geometry. Linear interpolation for point placement along path segments ensures that resampled points lie exactly on the original drawn trajectory without introducing smoothing artifacts.
