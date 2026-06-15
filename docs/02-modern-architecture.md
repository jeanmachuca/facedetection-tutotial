# Modern Architecture: Native Browser Face Detection

## Overview

Modern face detection uses standardized web APIs and pre-trained neural networks, enabling fast, accurate, and privacy-preserving detection without plugins.

## Core Technologies

### 1. WebRTC - Camera Access

Standardized by W3C (2012-present). Uses `navigator.mediaDevices.getUserMedia()`.

```javascript
const stream = await navigator.mediaDevices.getUserMedia({
    video: { width: { ideal: 1280 }, height: { ideal: 720 }, facingMode: 'user' },
    audio: false
});
videoElement.srcObject = stream;
```

**Advantages over Flash**: No plugin required, unified permission model, works on mobile (iOS 14.5+, Android 5.0+), better performance, direct GPU access, HTTPS security.

**Constraints**: HTTPS required (or localhost), user permission required, same-origin restriction.

### 2. Canvas API - Image Processing

```javascript
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
ctx.drawImage(video, 0, 0, width, height);
const imageData = ctx.getImageData(0, 0, width, height);
```

~1ms per 640x480 frame on modern hardware, GPU-accelerated in most browsers.

### 3. Web Workers - Parallel Processing

Keep main thread responsive during heavy computation. Detection runs in background thread while main thread stays at 60 FPS.

### 4. WebGL - GPU Acceleration

10-50x faster than CPU for matrix operations. Example: ResNet50 face detection - CPU: 500-800ms per frame, WebGL: 20-50ms per frame.

### 5. TensorFlow.js - ML Runtime

```javascript
import * as faceapi from 'face-api.js';

await faceapi.nets.tinyFaceDetector.loadFromUri('/models');
await faceapi.nets.faceLandmark68Net.loadFromUri('/models');
await faceapi.nets.faceRecognitionNet.loadFromUri('/models');

const detections = await faceapi
    .detectAllFaces(video)
    .withFaceLandmarks()
    .withFaceDescriptors();
```

**Available Models**:

| Model | Size | Latency | Accuracy | Use Case |
|-------|------|---------|----------|----------|
| TinyFaceDetector | 200 KB | 20ms | 85% | Lightweight, resource-constrained |
| MTCNN | 200 KB | 50ms | 92% | Medium accuracy, good for poses |
| SSD MobileNetV2 | 27 MB | 50ms | 98% | High accuracy, standard choice |
| BlazeFace | 160 KB | 15ms | 90% | Mobile optimized, very fast |

**Backends**: CPU (fallback), WebGL (GPU, most common), WebAssembly (near-native speed), WebGPU (newest, experimental).

### 6. face-api.js - High-Level Wrapper

**Built on**: TensorFlow.js + Pre-trained Models

**Provides**: Face Detection (bounding box + confidence), Facial Landmarks (68 points), Face Recognition (128-dimensional embeddings), Age/Gender/Expression Detection.

### 7. MediaPipe - Google's Alternative

Optimized for mobile phones and edge devices.

**vs face-api.js**:

| Aspect | face-api.js | MediaPipe |
|--------|-------------|-----------|
| Model Size | 30MB+ | 5-10MB |
| Inference Speed | 50-100ms | 10-30ms |
| Accuracy | 95%+ | 98%+ |
| Mobile | Works | Highly optimized |
| Setup | npm install | CDN |

## Data Flow

### Real-Time Detection Pipeline
```
Video Stream (30-60 FPS) → Canvas Capture → Web Worker (parallel)
    → TensorFlow.js + GPU (normalize, CNN inference)
        → Post-Processing (NMS, thresholding)
            → Main Thread (render boxes, update DOM)
```

### Training Data Migration
```
Legacy JPEG Images → Load in Browser (Canvas)
    → face-api.js Detection → Extract embeddings (128-dim)
        → JSON Array of Vectors → Storage (IndexedDB / Backend / JSON file)
```

## Comparison: Flash vs Modern

### Development Experience

| Aspect | Flash | Modern |
|--------|-------|--------|
| Setup Time | 30+ min (Flex SDK) | 5 min (npm) |
| Build Step | MXML → SWF compiler | None (or esbuild) |
| Debugging | Limited | Full browser DevTools |
| Hot Reload | No | Yes |
| TypeScript | No | Yes |
| Dependency Mgmt | Manual SWF libs | npm/package.json |
| Learning Curve | Steep | Moderate |

### Runtime Performance

| Operation | Flash | Modern |
|-----------|-------|--------|
| Cold Start | 2-3s | 500ms-2s |
| Model Load | Built-in | 5-30MB download |
| Frame Capture | 15-20ms | 5-10ms |
| Detection (30 FPS) | Achievable | Achievable |
| Memory | ~150MB | ~100MB |
| GPU Support | No | Yes (WebGL) |
| Mobile | Not available | Good |

### Accuracy

**Flash (Haar Cascade)**: 80-85%, issues with occlusion, rotation, scale variation.

**Modern (CNN-based)**: 95-98%, robust to +/-30 rotation, partial occlusion, scale variation. Pre-trained on millions of faces.

## Model Architecture Deep Dive

### Legacy: Haar Cascade
```
Input → Haar Features (2x2 rectangles: light-dark, edge, line)
    → Cascade Classifier (30 stages, each filtering more)
        → Output: (x, y, w, h) or null
```
Fast, no GPU needed, small model, but brittle with many false positives.

### Modern: Convolutional Neural Networks
```
Input Image (640x480 RGB)
    → Conv Layers (3x3, 32→64→128 filters, MaxPool)
        → Detection Head (YOLO-style: grid predictions, scores, boxes)
            → Post-Processing (NMS, thresholding)
                → Output: [(x1,y1,x2,y2,score), ...]
```
Accurate, robust, generalizes well, but larger model, GPU recommended.
