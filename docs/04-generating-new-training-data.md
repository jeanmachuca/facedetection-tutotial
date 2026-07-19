# Generating New Training Data

## Overview

There are three approaches to adding new training data, depending on whether you want to work with the legacy Flash system, the modern web app, or build a standalone pipeline.

---

## Approach 1: Add JPEG Images to the Legacy Dataset (Archival)

This keeps your data compatible with both repos but **cannot regenerate `trainedData.php`** (requires Flash Player).

### Steps

1. **Prepare your images**
   - Format: JPEG
   - Resolution: 640x480 or higher
   - File size under 200KB (matching the legacy `upload.php` limit)
   - Front-facing, good lighting, no motion blur

2. **Name them consistently**
   ```
   <PERSON_LABEL><NUMBER>.jpg
   ```
   Examples: `JD1.jpg`, `JD2.jpg`, `JD3.jpg` for a person labeled "JD"

3. **Add to the legacy repo**
   ```bash
   cp JD1.jpg JD2.jpg JD3.jpg \
     /path/to/facerecognitioninbrowser/faces/trainingSet/images/
   ```

4. **Update `trainedData.php`** — This file **cannot** be regenerated without Flash Player. The legacy app will still detect faces but won't recognize the new ones. The images are preserved for use with the modern pipeline.

### File naming convention reference

| Prefix | Person | Existing samples |
|--------|--------|-----------------|
| `AW` | Person AW | 3 |
| `NG` | Person NG | 5 |
| `OW` | Person OW | 13 |

Use a unique 2-3 letter prefix for each new person.

---

## Approach 2: Use the Modern Web App — Real-Time Capture (Recommended)

The `modern-face-detection-app-example` app supports real-time face training directly from the camera. No static images, no file management — just point, name, and capture.

### How It Works

The app captures face samples live from the webcam:

1. You enter a person's name
2. You click "Capture Face" to snapshot the current camera frame
3. The app runs face-api.js on the frame: detect face → extract 68 landmarks → generate 128-dim descriptor
4. The descriptor is stored in memory under the person's label
5. Repeat until you have at least 3 samples, then click "Save Person"
6. The live detection loop immediately recognizes saved people

### Step-by-step

#### 1. Start the app

```bash
cd /path/to/modern-face-detection-app-example
npx serve .
```

Open the URL in your browser (Chrome/Firefox/Edge recommended).

#### 2. Start the camera

Click **"Start Camera"** and grant webcam permission. You'll see the live video feed with a face detection overlay.

#### 3. Train a person

1. Type the person's name into the **"Person Name"** field
2. Position the person's face in the camera view
3. Click **"Capture Face"** — a thumbnail of the captured frame appears in the gallery
4. Repeat 2 more times (minimum 3 samples required)
5. Vary the angle slightly between captures for better recognition
6. Click **"Save Person"** — the person is now recognized in real time

#### 4. Repeat for more people

After saving, the name field clears. Enter the next person's name and repeat the capture process. The detection loop recognizes all saved people simultaneously.

### Under the Hood

When you click "Capture Face", this code runs:

```javascript
// 1. Grab current video frame onto offscreen canvas
const canvas = document.createElement('canvas');
canvas.width = videoElement.videoWidth;
canvas.height = videoElement.videoHeight;
canvas.getContext('2d').drawImage(videoElement, 0, 0);

// 2. Detect face and extract 128-dim descriptor
const descriptor = await detector.getDescriptor(canvas);

// 3. Store under the person's label
labeledDescriptors[label].push(descriptor);
```

Recognition uses Euclidean distance between the live descriptor and all stored descriptors:

```javascript
function recognize(descriptor, labeledDescriptors, threshold = 0.5) {
    for (const [name, descriptors] of Object.entries(labeledDescriptors)) {
        for (const ref of descriptors) {
            const distance = faceapi.euclideanDistance(descriptor, ref);
            if (distance < threshold) return { name, confidence: 1 - distance };
        }
    }
    return null;
}
```

### Advantages Over Static Image Training

| Aspect | Static (old) | Real-Time Capture (new) |
|--------|-------------|------------------------|
| Setup | Copy JPEG files, edit code | Just open the app |
| Speed | Process 21 images on load | Capture 3-5 frames in seconds |
| Flexibility | Fixed dataset | Train anyone, anywhere |
| Accuracy | Depends on image quality | Depends on camera quality |
| Persistence | Files on disk | In-memory (session only) |

### Tips for Better Recognition

- Capture in the same lighting you'll use for recognition
- Vary the angle slightly (front, slight left, slight right)
- Ensure the face is well-lit and clearly visible
- Capture at least 5 samples per person for more reliable matching
- Use SSD MobileNetV1 (default) for best accuracy

### Limitations

- Training data is session-only (lost on page reload)
- No export/import mechanism built in (see [Approach 3](#approach-3-build-a-standalone-training-pipeline-advanced) for persistence)
- Minimum 3 samples per person enforced

---

## Approach 3: Build a Standalone Training Pipeline (Advanced)

For batch-processing large datasets or generating reusable JSON embedding files, use a Node.js script.

### Setup

```bash
mkdir training-pipeline && cd training-pipeline
npm init -y
npm install @vladmandic/face-api @tensorflow/tfjs-node canvas
```

### Pipeline script

Create `generate-embeddings.js`:

```javascript
const faceapi = require('@vladmandic/face-api');
const tf = require('@tensorflow/tfjs-node');
const { createCanvas, loadImage } = require('canvas');
const fs = require('fs');
const path = require('path');

const MODEL_PATH = './node_modules/@vladmandic/face-api/model';

async function main() {
    // Load models
    await faceapi.nets.ssdMobilenetv1.loadFromDisk(MODEL_PATH);
    await faceapi.nets.faceLandmark68Net.loadFromDisk(MODEL_PATH);
    await faceapi.nets.faceRecognitionNet.loadFromDisk(MODEL_PATH);

    const imagesDir = process.argv[2] || './images';
    const outputFile = process.argv[3] || 'embeddings.json';
    const files = fs.readdirSync(imagesDir).filter(f => /\.(jpg|jpeg|png)$/i.test(f));

    const database = { version: '1.0', model: 'ssdMobilenetv1', faces: [] };

    for (const file of files) {
        // Extract label from filename: e.g., "JD1.jpg" -> "JD"
        const label = file.match(/^([A-Z]+)/)?.[1] || 'unknown';

        const image = await loadImage(path.join(imagesDir, file));
        const canvas = createCanvas(image.width, image.height);
        const ctx = canvas.getContext('2d');
        ctx.drawImage(image, 0, 0);

        const detection = await faceapi
            .detectSingleFace(canvas, new faceapi.SsdMobilenetv1Options({ minConfidence: 0.5 }))
            .withFaceLandmarks()
            .withFaceDescriptor();

        if (detection) {
            database.faces.push({
                file,
                label,
                descriptor: Array.from(detection.descriptor),
                imageWidth: image.width,
                imageHeight: image.height,
                detectionScore: detection.detection.score,
            });
            console.log(`✓ ${file} → ${label} (score: ${detection.detection.score.toFixed(2)})`);
        } else {
            console.warn(`✗ ${file} → no face detected`);
        }
    }

    fs.writeFileSync(outputFile, JSON.stringify(database, null, 2));
    console.log(`\nSaved ${database.faces.length} embeddings to ${outputFile}`);
}

main().catch(console.error);
```

### Usage

```bash
# Place images in ./images/ with names like JD1.jpg, JD2.jpg, SM1.jpg, etc.
node generate-embeddings.js ./images embeddings.json
```

### Output format

```json
{
    "version": "1.0",
    "model": "ssdMobilenetv1",
    "faces": [
        {
            "file": "JD1.jpg",
            "label": "JD",
            "descriptor": [0.123, -0.045, 0.678, ...],
            "imageWidth": 640,
            "imageHeight": 480,
            "detectionScore": 0.97
        }
    ]
}
```

### Using the exported JSON in the modern app

```javascript
// Load pre-computed embeddings
const response = await fetch('embeddings.json');
const data = await response.json();

const labeledDescriptors = {};
for (const face of data.faces) {
    if (!labeledDescriptors[face.label]) {
        labeledDescriptors[face.label] = [];
    }
    labeledDescriptors[face.label].push(new Float32Array(face.descriptor));
}

detector.setTrainingData(labeledDescriptors);
```

---

## Best Practices for Collecting Training Images

### Quantity

| Scenario | Samples per person | Notes |
|----------|------------------|-------|
| Minimal (verification) | 3-5 | High false-positive rate |
| Good (identification) | 10-20 | Reliable for most use cases |
| Production | 50-100 | Handles varied conditions |
| Enterprise | 200+ | Multiple angles, lighting, expressions |

### Image Quality Guidelines

| Requirement | Recommended | Minimum |
|------------|-------------|---------|
| Resolution | 1280x720 | 640x480 |
| Face size | >300px wide | >100px wide |
| Lighting | Even, diffused | No extreme shadows |
| Background | Plain, solid color | No cluttered backgrounds |
| Expression | Neutral to slight smile | Natural |
| Angle | Frontal (+/- 15°) | Frontal (+/- 30°) |

### Variety

For each person, collect images with:
- **Lighting**: bright, dim, side-lit
- **Angle**: front, slight left, slight right
- **Expression**: neutral, smiling, serious
- **Accessories**: with/without glasses if applicable
- **Time**: different days to capture natural variation

### Privacy & Ethics

- **Always** obtain written consent before collecting someone's face data
- Clearly explain how the data will be used
- Allow people to request deletion of their data
- Store only face descriptors (not raw images) in production
- Never share training data without explicit permission
- Consider anonymizing data when possible

### File Management

```
training-data/
├── images/
│   ├── JD1.jpg       # Person JD, sample 1
│   ├── JD2.jpg       # Person JD, sample 2
│   ├── SM1.jpg       # Person SM, sample 1
│   └── ...
├── embeddings.json   # Generated by pipeline
└── manifest.csv      # Optional: metadata for each image
```

Example `manifest.csv`:
```csv
file,label,date,lighting,angle,notes
JD1.jpg,JD,2026-06-15,bright,frontal,studio lighting
JD2.jpg,JD,2026-06-15,dim,slight-left,evening
SM1.jpg,SM,2026-06-16,bright,frontal,outdoor
```

---

## Quick Reference: Which Approach Should I Use?

| Your goal | Use this approach |
|-----------|-----------------|
| Archive images for historical reference | Approach 1 (add JPEGs to legacy repo) |
| Train new people quickly in the browser | Approach 2 (real-time camera capture) |
| Batch-process 50+ images | Approach 3 (Node.js pipeline) |
| Generate a portable JSON embedding file | Approach 3 (Node.js pipeline) |
| Deploy to production | Approach 3 + JSON loading in app |

---

## Troubleshooting

### Face not detected in my new image
- Ensure the face is clearly visible (at least 100x100 pixels)
- Check lighting — avoid backlighting or harsh shadows
- The face should be facing the camera (within 30°)
- Try a different image with better quality

### Poor recognition accuracy with new people
- Add more training samples (aim for 10+ per person)
- Include variety in lighting and angles
- Lower the match threshold in the UI (try 0.4 instead of 0.5)
- Ensure training images are high resolution

### Batch script fails
- Install system dependencies for `node-canvas`:
  ```bash
  # Ubuntu/Debian
  sudo apt-get install build-essential libcairo2-dev libpango1.0-dev libjpeg-dev libgif-dev librsvg2-dev
  # macOS
  brew install pkg-config cairo pango libpng jpeg giflib librsvg
  ```
- Ensure TensorFlow backend matches your platform (`@tensorflow/tfjs-node` requires Node.js 12+)
