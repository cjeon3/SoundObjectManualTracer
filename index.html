# Sound Object Contour Tracer

**Version 4.1** | January 13, 2026

## Purpose

This application digitizes hand-drawn sound object contours from auditory spatial perception experiments. Participants draw shapes representing their perception of sound objects at various frequencies, and this tool extracts coordinate data from those drawings for subsequent acoustic analysis.

## Application Access

The application runs in-browser at: https://cjeon3.github.io/SoundObjectManualTracer/

## Coordinate System

The drawing canvas implements a 1000 × 1000 pixel grid mapped to a 20 × 20 unit coordinate system ranging from -10 to +10 on both axes. The scale factor is 50 pixels per unit, with the origin at canvas center (500, 500). A reference circle of radius 3 units is displayed at the origin.

Coordinate transformations between pixel and unit space:

```
x_unit = (x_pixel - 500) / 50
y_unit = (500 - y_pixel) / 50
```

This coordinate system matches the original drawing application used by participants, ensuring 1:1 alignment between source images and traced contours.

## Experimental Frequencies

The application supports six frequency conditions used in the study:

| Frequency | Presentation Level |
|-----------|-------------------|
| 62.5 Hz   | 100 dB            |
| 125 Hz    | 90 dB             |
| 250 Hz    | 85 dB             |
| 500 Hz    | 80 dB             |
| 1000 Hz   | 80 dB             |
| 2000 Hz   | 80 dB             |

## Data Input

### Image Upload

Source images are uploaded as PNG files. The application accepts individual files or ZIP archives containing nested folder structures. When using ZIP upload, the folder hierarchy determines participant and trial assignments:

```
ParticipantName/
    Trial1/
        frequency_image.png
    Trial2/
        frequency_image.png
```

Frequency information is extracted from filenames (e.g., "62.5Hz_100dB.png").

### Data Import

Previously exported JSON files can be imported to resume tracing sessions or combine datasets from multiple tracers.

## Tracing Procedure

### Contour Types

Two contour types are traced for each frequency condition:
- **Red contours**: 0° Interaural Phase Difference (IPD)
- **Blue contours**: 90° right-leading Interaural Phase Difference (IPD)

### Drawing Controls

- Brush size: Adjustable from 1-10 pixels
- Drawing toggle: Enables or disables stroke input
- Undo/Redo: Maintains complete edit history per frequency
- Grid overlay: Toggleable coordinate reference grid

### Display Parameters

- **Image opacity**: Controls visibility of the underlying participant drawing (0-100%)
- **Shape opacity**: Controls transparency of previously traced contours (0-100%)
- **Resample rate**: Number of interpolated points for shape averaging (500-10,000)

## Composite Image Generation

The application generates cross-participant composite images for each frequency. This process collects all traced contours across participants and trials, then computes an average contour using simple coordinate averaging after resampling all shapes to uniform point counts.

### Frequency Selection

Composites can be generated for all frequencies simultaneously or for a selected subset. The selection modal displays shape counts per frequency to indicate data availability.

### Output Statistics

Exported composite images include:
- Individual traced contours at reduced opacity
- Averaged contour at full opacity
- Centroid marker for each color
- Sample size (N), centroid coordinates, and enclosed area
- Centroid shift distance between red and blue averages

## Data Export

### Local Export

- **CSV**: Flattened coordinate table with columns for participant, trial, frequency, color, shape index, point index, x, and y
- **JSON**: Complete hierarchical data structure preserving all metadata and settings

### Cloud Storage

The application integrates with Google Sheets for collaborative data collection. Multiple tracers save progress independently under unique identifiers, and datasets can be merged for analysis.

## Google Apps Script Configuration

Cloud storage requires deploying the provided Apps Script to a Google Sheets spreadsheet.

### Setup Procedure

1. Create a new Google Sheets spreadsheet
2. Open the script editor (Extensions > Apps Script)
3. Replace the default code with the contents of `google_cloud_collab_script.gs`
4. Update the `SHEET_ID` constant with your spreadsheet ID (found in the URL)
5. Deploy as a web application with "Anyone" access
6. Copy the deployment URL into the application's configuration field

### Data Structure

The script creates four sheets:

| Sheet | Contents |
|-------|----------|
| TracerProgress | Primary storage with tracer metadata and JSON-encoded coordinate data |
| CoordinateData | Denormalized coordinate table for direct spreadsheet analysis |
| Summary | Aggregated shape counts per participant, trial, and frequency |
| ActivityLog | Timestamped record of save and load operations |

### Large Dataset Handling

Google Sheets limits cell contents to 50,000 characters. The script automatically partitions large JSON payloads across multiple cells and reassembles them on retrieval. This chunking is transparent to the user.

## File Manifest

```
index.html                      Main application
google_cloud_collab_script.gs   Google Apps Script for cloud storage
README.md                       Documentation
```

## Browser Requirements

The application requires a modern browser with HTML5 Canvas support. Tested on current versions of Chrome, Firefox, Safari, and Edge.
