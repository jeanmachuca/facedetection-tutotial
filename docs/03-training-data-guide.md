# Training Data Guide: Migrating from Flash to Modern

## Legacy Dataset Analysis

The original `facerecognitioninbrowser` project contains:

- **23 JPEG images** across 3 subjects
- **trainedData.php** (493 KB) - proprietary serialized format

The binary `trainedData.php` cannot be directly used in modern systems. The original JPEG images, however, are perfectly usable.

## Migration Process

### Step 1: Collect Original Images

The images from `faces/trainingSet/images/` are standard JPEG files that can be used directly. Copy them to your modern project.

### Step 2: Process with Modern Pipeline

```javascript
// 1. Load images
const img = new Image();
img.src = 'images/AW1.jpg';
await img.decode();

// 2. Detect face and extract embedding
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');
ctx.drawImage(img, 0, 0);
const detection = await faceapi
    .detectSingleFace(canvas)
    .withFaceLandmarks()
    .withFaceDescriptor();

// 3. Store as JSON
const embedding = detection.descriptor; // Float32Array(128)
```

### Step 3: Generate Embeddings JSON

```json
{
  "AW": [
    { "file": "AW1.jpg", "descriptor": [0.123, 0.456, ...] },
    { "file": "AW2.jpg", "descriptor": [0.125, 0.458, ...] }
  ],
  "NG": [...],
  "OW": [...]
}
```

### Step 4: Use in Recognition

```javascript
function findMatch(descriptor, labeledDescriptors, threshold = 0.6) {
    for (const [name, descriptors] of Object.entries(labeledDescriptors)) {
        for (const ref of descriptors) {
            const distance = faceapi.euclideanDistance(descriptor, ref);
            if (distance < threshold) return { name, distance };
        }
    }
    return null;
}
```

## Modern Data Format

Use JSON for structured storage of face descriptors:

```typescript
interface FaceDescriptor {
    file: string;
    descriptor: number[];      // 128-dimensional vector
    label: string;             // Person identifier
    createdAt?: string;
    metadata?: {
        imageWidth?: number;
        imageHeight?: number;
        detectionScore?: number;
    };
}

interface FaceDatabase {
    version: string;
    model: string;             // e.g., "face-api.js v0.22"
    faces: FaceDescriptor[];
}
```

## Best Practices

### Data Collection
- Collect 5-10 images per person
- Vary lighting conditions
- Vary angles (front, slight left/right)
- Ensure good image quality (>640x480)
- Avoid motion blur

### Processing
- Use face detection confidence threshold >0.8
- Validate that landmarks are detected (68 points)
- Normalize embeddings (L2 normalization)
- Store multiple samples per person for better matching

### Matching
- Use cosine similarity or Euclidean distance
- Typical thresholds: 0.4-0.6 (lower = stricter)
- Consider k-NN or SVM for multiple samples
- Implement confidence-based rejection

## Performance Tips

| Tip | Impact |
|-----|--------|
| Use TinyFaceDetector for real-time | 5x faster, slightly less accurate |
| Cache embeddings in IndexedDB | No re-processing on reload |
| Use Web Workers for detection | No UI jank |
| Load models on demand | Faster initial page load |
| Resize input to 640x480 | 3x faster than 1280x720 |

## Security Considerations

- **Privacy**: All processing happens client-side; no images leave the browser
- **Storage**: Use IndexedDB for persistent storage; don't store raw images if not needed
- **Permissions**: Request camera access only when needed; explain why
- **HTTPS**: Required for WebRTC; also protects data in transit

## Troubleshooting

### Common Issues

**Face not detected**: Ensure good lighting; face should be front-facing and at least 100x100 pixels in the frame.

**Poor matching accuracy**: Add more training samples per person; vary lighting and angles; adjust threshold.

**Low FPS**: Use TinyFaceDetector instead of SSD MobileNetV2; reduce canvas resolution; enable WebGL backend.

**Model loading fails**: Check CDN URLs; ensure models/ directory is accessible; verify browser compatibility.
