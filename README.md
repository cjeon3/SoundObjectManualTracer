# Sound Object Contour Tracer

## Technical Documentation

### UCI Hearing & Speech Lab

Version 3.0

---

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Coordinate System Specification](#coordinate-system-specification)
4. [Data Structure](#data-structure)
5. [Application Layout](#application-layout)
6. [Processing Pipeline](#processing-pipeline)
7. [Google Sheets Integration](#google-sheets-integration)
8. [Data Export Format](#data-export-format)
9. [Setup Instructions](#setup-instructions)
10. [File Format Requirements](#file-format-requirements)

---

## Overview

The Sound Object Contour Tracer is a web-based application designed for manual digitization of participant-drawn shapes from auditory spatial perception experiments. The application enables researchers to trace red (in-phase) and blue (out-of-phase) contours from participant drawings, storing precise coordinate data for subsequent statistical analysis.

The tool supports collaborative workflows through cloud-based data synchronization via Google Sheets, allowing multiple researchers to contribute tracing work to a shared dataset while maintaining individual progress tracking.

---

## System Architecture

The application consists of two primary components:

1. **Client Application**: A single-file HTML application utilizing HTML5 Canvas for rendering and user interaction, with JavaScript handling all computation and data management.

2. **Cloud Backend**: A Google Apps Script web application that provides RESTful API endpoints for data persistence, retrieval, and multi-user collaboration.

### Technology Stack

- HTML5 Canvas API for drawing and image rendering
- JavaScript (ES6+) for application logic
- JSZip library for compressed file handling
- FileSaver.js for local file exports
- Google Apps Script for cloud storage
- Google Sheets as the database layer

---

## Coordinate System Specification

The application employs a Cartesian coordinate system identical to that used in the original participant drawing application. This ensures precise 1:1 alignment between the background PNG images and the tracing canvas.

### Canvas and Grid Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| Canvas Size | 1000 x 1000 pixels | Physical canvas dimensions |
| Unit Range | -10 to +10 | Coordinate range on both axes |
| Scale Factor | 50 pixels per unit | Conversion ratio (1000 pixels / 20 units) |
| Origin | (500, 500) pixels | Canvas center point |
| Reference Circle | 3 unit radius (150 pixels) | Represents participant head position |
| Grid Lines | 1 unit spacing (50 pixels) | Visual reference grid |
| Brush Width | 5 pixels | Default stroke width matching original application |

### Original Drawing Application Specifications

The participant drawing application used the following identical specifications:

- Canvas: 1000 x 1000 pixels
- Coordinate grid: 20 x 20 units (-10 to +10 on each axis)
- Scale factor: 50 pixels per unit
- Background circle radius: 3 units (representing the listener's head)
- Brush size: 5 pixels
- Red color (#ef4444): In-phase condition (0 degrees IPD)
- Blue color (#3b82f6): Out-of-phase condition (90 degrees IPD)

### Coordinate Conversion

The transformation between canvas pixels and unit coordinates follows these formulas:

```
Unit X = (Pixel X - 500) / 50
Unit Y = (500 - Pixel Y) / 50
```

Note that the Y-axis is inverted to convert from canvas coordinates (origin at top-left) to standard Cartesian coordinates (origin at center, positive Y upward).

### Color Specification

| Phase Condition | Color | Hex Value | RGB |
|-----------------|-------|-----------|-----|
| In-phase (0 degrees IPD) | Red | #ef4444 | (239, 68, 68) |
| Out-of-phase (90 degrees IPD) | Blue | #3b82f6 | (59, 130, 246) |

---

## Data Structure

### Hierarchical Organization

All traced data is organized in a three-level hierarchy:

```
allData
  [Participant Name]
    [Trial Identifier]
      [Frequency Key]
        red: Array of shape arrays
        blue: Array of shape arrays
        redAvg: Average shape array or null
        blueAvg: Average shape array or null
```

### Shape Representation

Each shape is represented as an array of coordinate objects:

```javascript
[
  { x: -2.5, y: 3.2 },
  { x: -2.3, y: 3.5 },
  { x: -2.0, y: 3.7 },
  // ... additional points
]
```

Coordinates are stored in unit space with values ranging from -10 to +10 on both axes.

### Frequency Key Format

Frequency keys follow the pattern: `[Frequency]Hz_[Loudness]dB`

Example: `62.5Hz_100dB`, `500Hz_80dB`

The loudness value is retained for reference but is not used in analysis.

---

## Application Layout

### Interface Sections

The application interface is organized into the following sections, presented in vertical order:

#### 1. Header Section

Displays application title and institutional affiliation.

#### 2. Upload Section

Provides three input methods:
- **Upload ZIP**: Accepts compressed archives containing participant image files
- **Single PNG**: Accepts individual image files for manual entry
- **Import JSON**: Accepts previously exported data for continued work

An upload log displays processing status and diagnostic information during file import.

#### 3. Participant Tabs

Horizontal tab bar displaying all participants found in the uploaded data. Selecting a participant tab loads their associated trials.

#### 4. Trial Tabs

Horizontal tab bar displaying trials (Trial 1, Trial 2, Trial 3) for the selected participant. Trial assignment is determined by folder number sorting (described in Processing Pipeline).

#### 5. Frequency Tabs

Horizontal tab bar displaying available frequencies for the selected participant and trial. The application processes frequencies in the range of 62.5 Hz to 2000 Hz:
- 62.5 Hz
- 125 Hz
- 250 Hz
- 500 Hz
- 1000 Hz
- 2000 Hz

Frequencies outside this range are excluded from processing.

#### 6. Canvas Section

The primary drawing area displaying:
- Background participant image at reduced opacity (default 10%)
- Reference circle indicating head position (3 unit radius)
- Previously traced shapes at full opacity
- Average shapes when computed (displayed with darker colors and thicker lines)
- Coordinate grid overlay (1 unit spacing)

#### 7. Drawing Controls

- **Color Selection**: Toggle between red and blue drawing modes
- **Brush Size**: Adjustable from 1 to 10 pixels (default: 5 pixels, matching the original drawing application used by participants)
- **Resample Rate**: Points per shape for averaging calculations (range: 500-10000, default: 1000)
- **Image Opacity**: Visibility of background participant image (range: 0-100%, default: 10%)

Note: Traced shapes are always rendered at full opacity to ensure accurate visual representation of the contours.

#### 8. Action Buttons

- **Undo/Redo**: Navigate through drawing history
- **Clear Current Color**: Remove all shapes of selected color for current frequency
- **Clear All**: Remove all shapes for current frequency
- **Compute Average**: Generate average shape from traced shapes

#### 9. Cloud Collaboration Section

- **Tracer Name**: Identifier for the current user
- **Apps Script URL**: Endpoint for Google Sheets integration
- **Save to Cloud**: Upload current progress to Google Sheets
- **Load from Cloud**: Retrieve previously saved progress
- **Auto-Save Toggle**: Enable automatic saving after each shape

#### 10. Export Section

- **Export JSON**: Download complete dataset as JSON file
- **Export Current PNG**: Save current canvas view as image

---

## Processing Pipeline

### Stage 1: File Upload and Extraction

When a ZIP file is uploaded, the application performs the following operations:

1. **Archive Parsing**: The JSZip library extracts all files from the compressed archive.

2. **Nested ZIP Handling**: If the archive contains additional ZIP files, these are recursively extracted.

3. **PNG Identification**: All PNG image files are identified for processing.

### Stage 2: Folder Structure Analysis

The application expects folders named according to the pattern:

```
[Participant Name]_[Folder Number]_[Optional Description]
```

Examples:
- `Jiaxin L_3_Drawings`
- `Jiaxin L_5_Drawings 1`
- `Jiaxin L_11_Drawings 2`

The parsing algorithm:
1. Splits the folder name by underscore characters
2. Extracts the participant name (all text before the first numeric segment)
3. Extracts the folder number (first numeric segment after participant name)
4. Ignores any text following the folder number

### Stage 3: Trial Mapping

For each participant, the application:

1. Collects all folder numbers associated with that participant
2. Sorts folder numbers in ascending numerical order
3. Maps sorted positions to trial numbers

Example:
```
Participant: Jiaxin L
Folder numbers found: [11, 3, 5]
Sorted: [3, 5, 11]
Mapping: 
  Folder 3  -> Trial 1
  Folder 5  -> Trial 2
  Folder 11 -> Trial 3
```

### Stage 4: Filename Parsing

Image filenames follow the pattern:

```
[Participant Name]_[Frequency]Hz_[Loudness]dB.png
```

Example: `Jiaxin L_62.5Hz_100dB.png`

The parsing algorithm extracts:
- **Participant Name**: Text preceding the frequency pattern
- **Frequency**: Numeric value followed by "Hz"
- **Loudness**: Numeric value followed by "dB" (stored but not used)

### Stage 5: Image Loading and Tab Construction

For each valid image file:

1. The image is loaded into memory
2. A unique key is generated: `[Participant]|[Trial]|[Frequency Key]`
3. The tab hierarchy is constructed based on discovered participants, trials, and frequencies
4. Data structures are initialized for shape storage

### Stage 6: Manual Tracing

The user performs manual tracing by:

1. Selecting the appropriate participant, trial, and frequency tabs
2. Selecting the drawing color (red or blue)
3. Drawing contours on the canvas using mouse or touch input
4. Points are captured continuously during drawing and stored as coordinate arrays

### Stage 7: Coordinate Resampling

When computing average shapes, all traced shapes are resampled to a uniform point count:

1. The total path length of each shape is calculated
2. Points are interpolated at equal arc-length intervals
3. The resample rate parameter determines the number of output points

### Stage 8: Average Shape Computation

The averaging algorithm:

1. Resamples all shapes of the same color to equal point counts
2. For each point index, calculates the mean X and Y coordinates across all shapes
3. Stores the resulting average shape in the data structure

---

## Google Sheets Integration

### Authentication and Configuration

The Google Apps Script backend requires:

1. A Google Sheet created to store data
2. The Sheet ID extracted from the URL
3. The Apps Script deployed as a web application with appropriate access permissions

### API Endpoints

#### Save Progress (POST)

Stores the complete dataset for a tracer:

```
POST [Script URL]
Content-Type: application/json

{
  "action": "saveProgress",
  "tracerName": "Researcher Name",
  "allData": { ... },
  "settings": {
    "resampleRate": 1000,
    "imageOpacity": 10,
    "brushSize": 5
  }
}
```

#### Load Progress (GET)

Retrieves a tracer's saved dataset:

```
GET [Script URL]?action=loadProgress&tracerName=Researcher%20Name
```

#### List Tracers (GET)

Returns all tracers with statistics:

```
GET [Script URL]?action=listTracers
```

#### Merge All Data (GET)

Combines data from all tracers:

```
GET [Script URL]?action=mergeAll
```

### Sheet Structure

The Google Apps Script creates four sheets:

#### TracerProgress Sheet

Stores complete JSON data per tracer for exact reconstruction:

| Column | Description |
|--------|-------------|
| TracerName | Identifier for the researcher |
| SavedAt | ISO timestamp of last save |
| Version | Data format version |
| ParticipantCount | Number of participants |
| TrialCount | Number of participant-trial combinations |
| TotalShapes | Total traced and average shapes |
| Settings_JSON | JSON string of application settings |
| AllData_JSON | JSON string of complete dataset |

#### CoordinateData Sheet

Flattened coordinate data for direct analysis:

| Column | Description |
|--------|-------------|
| TracerName | Researcher identifier |
| Participant | Participant name |
| Trial | Trial identifier (e.g., "Trial 1") |
| Frequency_Hz | Frequency value |
| Color | "Red" or "Blue" |
| ShapeType | "Traced" or "Average" |
| ShapeNumber | Index of shape (1-based) |
| PointIndex | Index of point within shape (1-based) |
| X | X coordinate in unit space |
| Y | Y coordinate in unit space |

#### Summary Sheet

Overview of tracing progress:

| Column | Description |
|--------|-------------|
| Participant | Participant name |
| Trial | Trial identifier |
| Frequency | Frequency key |
| RedShapes | Count of red traced shapes |
| BlueShapes | Count of blue traced shapes |
| RedAvg | "Yes" or "No" |
| BlueAvg | "Yes" or "No" |
| TotalPoints | Sum of all coordinate points |
| LastUpdatedBy | Tracer who last modified |
| LastUpdatedAt | Timestamp of last modification |

#### ActivityLog Sheet

Audit trail of all operations:

| Column | Description |
|--------|-------------|
| Timestamp | ISO timestamp |
| TracerName | Researcher identifier |
| Action | Operation type (SAVE, LOAD, MERGE) |
| Details | Description of operation |

---

## Data Export Format

### JSON Export

The exported JSON file contains the complete application state:

```json
{
  "exportedAt": "2024-01-15T10:30:00.000Z",
  "version": "3.0",
  "settings": {
    "resampleRate": 1000,
    "imageOpacity": 10,
    "brushSize": 5
  },
  "allData": {
    "Jiaxin L": {
      "Trial 1": {
        "62.5Hz_100dB": {
          "red": [
            [{"x": -2.5, "y": 3.2}, {"x": -2.3, "y": 3.5}]
          ],
          "blue": [
            [{"x": 1.2, "y": -2.8}, {"x": 1.5, "y": -3.0}]
          ],
          "redAvg": [{"x": -2.4, "y": 3.35}],
          "blueAvg": null
        }
      }
    }
  }
}
```

### PNG Export

The current canvas view can be exported as a PNG image file, preserving:
- Background reference circle
- All traced shapes at full opacity
- Average shapes if computed
- Coordinate grid overlay

---

## Setup Instructions

### Google Apps Script Configuration

1. Navigate to Google Sheets and create a new spreadsheet.

2. Copy the spreadsheet ID from the URL:
   ```
   https://docs.google.com/spreadsheets/d/[SPREADSHEET_ID]/edit
   ```

3. Navigate to script.google.com and create a new project.

4. Replace the contents of Code.gs with the provided Google Apps Script code.

5. Locate the configuration section and set the spreadsheet ID:
   ```javascript
   const SPREADSHEET_ID = 'your-spreadsheet-id-here';
   ```

6. Save the project.

7. Select Deploy from the menu, then New deployment.

8. Configure the deployment:
   - Select type: Web app
   - Execute as: Me
   - Who has access: Anyone

9. Click Deploy and authorize the application when prompted.

10. Copy the provided Web App URL.

### Application Configuration

1. Open the Contour Tracer HTML file in a web browser.

2. Enter your name in the Tracer Name field.

3. Paste the Web App URL in the Apps Script URL field.

4. These settings are stored in browser local storage for subsequent sessions.

---

## File Format Requirements

### ZIP Archive Structure

```
Archive.zip
  [Participant]_[FolderNum]_[Description]/
    [Participant]_[Frequency]Hz_[Loudness]dB.png
    [Participant]_[Frequency]Hz_[Loudness]dB.png
    ...
  [Participant]_[FolderNum]_[Description]/
    ...
```

### Image Specifications

| Property | Requirement |
|----------|-------------|
| Format | PNG |
| Dimensions | 1000 x 1000 pixels (recommended) |
| Color Space | RGB |
| Coordinate Grid | Must match application coordinate system |

Images of different dimensions will be scaled to fit the canvas while maintaining aspect ratio, though this may affect coordinate accuracy.

### Filename Pattern

```
[Participant Name]_[Frequency]Hz_[Loudness]dB.png
```

- Participant Name: May contain letters, numbers, and spaces
- Frequency: Numeric value (decimals allowed, e.g., 62.5)
- Loudness: Numeric value (ignored by application)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024 | Initial release |
| 2.0 | 2024 | Added cloud collaboration features |
| 3.0 | 2025 | Improved folder parsing, trial mapping, enhanced documentation |

---

## References

This application was developed for the Sound Object Phenomenon study conducted at the UCI Hearing and Speech Lab. The study investigates auditory spatial perception through participant drawings representing perceived sound object shapes at various frequencies under different binaural phase conditions.

---

## Contact

UCI Hearing and Speech Lab
University of California, Irvine
