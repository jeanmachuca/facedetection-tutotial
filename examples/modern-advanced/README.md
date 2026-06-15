# Modern Advanced Example

A production-ready face recognition app with WebRTC + face-api.js + training data migration.

See the **[modern-face-detection-app-example](https://github.com/jeanmachuca/modern-face-detection-app-example)** repository for the complete working implementation.

## Features

- Real-time face detection + recognition
- Legacy training data import
- JSON-based face database
- Multiple model support (TinyFaceDetector, SSD MobileNetV2)
- Web Workers for parallel processing
- IndexedDB storage for face embeddings
- Performance monitoring (FPS, latency)
- Responsive UI

## Architecture

```
index.html → src/app.js (main controller)
    → src/camera.js (WebRTC setup)
    → src/detection.js (face-api.js wrapper)
    → src/training.js (legacy data migration)
    → src/database.js (IndexedDB storage)
```
