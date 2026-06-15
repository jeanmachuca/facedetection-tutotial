# Legacy Architecture: Flash-based Face Detection

## Overview

The original `facerecognitioninbrowser` project represents the state-of-the-art in web-based face detection during the Flash era (2005-2020).

## Key Components

### 1. XHTML Frontend

**File**: `FaceRecognizerForBrowserUsingCamera.html`

Embeds Flash player using SWFObject, provides fallback for Flash-disabled browsers, handles browser history.

**Browser Compatibility**: IE 6+, Firefox 3.6+, Chrome 1+, Safari 3+. Requires Flash Player 11.1.0 or higher.

### 2. Adobe Flex Application

**File**: `FaceRecognizerForBrowserUsingCamera.swf` (compiled)

**Technology Stack**:
- Language: ActionScript 3 (compiled to bytecode)
- Framework: Adobe Flex 4.6
- UI Library: Spark framework
- Dependencies: framework_4.6.0.23201.swf (542 KB), spark_4.6.0.23201.swf (776 KB), mx_4.6.0.23201.swf (528 KB), rpc_4.6.0.23201.swf (209 KB), textLayout_2.0.0.232.swf (312 KB)

**Responsibilities**: Camera permission handling, real-time video stream processing (~30 FPS), face detection via FaceRecognitionLib, UI for results, communication with PHP backend, training data serialization.

### 3. Face Recognition Library

**Library**: FaceRecognitionLib (AS3)

**Capabilities**: Haar Cascade-based face detection, classical feature extraction, Euclidean distance face comparison, training via image upload.

**Algorithm Details**: Uses Haar Cascade classifiers (Viola-Jones), ~50-200 dimensional feature vectors, no deep learning, accuracy ~80-85% under ideal conditions.

### 4. Backend Storage

**Technology**: PHP + File System

**getTrainingSetData.php**: Scans images/ directory, returns XML with image metadata.

**upload.php**: Handles image uploads, validates type via `exif_imagetype()`, enforces max 10 files, maintains most-recent list, max file size 200KB.

**trainedData.php**: ~493 KB binary file containing serialized trained model data. PHP-serialized AS3 objects or raw feature vectors. **Not portable** to modern systems.

## Data Flow

### Training Phase
```
1. User selects image from disk
2. Flash app sends to upload.php
3. PHP validates and stores in /images
4. Flash app calls getTrainingSetData.php (XML response)
5. Flash loads images and displays in UI
6. User clicks "Train" button
7. FaceRecognitionLib extracts features from image
8. Features serialized and sent to backend
9. Backend stores in trainedData.php
```

### Detection Phase
```
1. Flash app accesses webcam via Camera API
2. Continuous frame capture at ~30 FPS
3. Convert frame to BitmapData
4. FaceRecognitionLib.detectFaces()
5. If face detected: extract features, compare against trainedData, return match + confidence
6. Display results in Flex UI (bounding boxes, match name, confidence, statistics)
```

## Why Flash?

Historical context - before WebRTC (2013), Flash was the only viable option for:
- Camera access via proprietary Flash Camera API
- Performance via ActionScript 3 JIT compilation
- Rich UI via Flex framework
- Cross-browser compatibility via single SWF file

## Limitations

- Plugin dependency (users must install Flash Player)
- Security concerns (frequent vulnerabilities)
- Haar Cascade accuracy ~80-85%, struggles with occlusion and extreme angles
- No mobile support (iOS never supported Flash)
- Complex build toolchain (MXML → SWF)
- Proprietary, non-portable format

## Training Data Format

**Input**: JPEG Images
```
faces/trainingSet/images/
├── AW1.jpg - AW3.jpg  (Person AW, 3 samples)
├── NG1.jpg - NG5.jpg  (Person NG, 5 samples)
└── OW1.jpg - OW13.jpg (Person OW, 13 samples)
Total: 23 images, ~750 KB
```

**Output**: `trainedData.php` (493,250 bytes, binary PHP-serialized or raw vectors)

## Modern Lessons from Flash

### What Flash Got Right
- Cross-browser consistency (single implementation works everywhere)
- Real-time performance (30 FPS video processing)
- Hardware integration (camera, microphone)
- Rich developer ecosystem

### What Flash Got Wrong
- Proprietary format (SWF) - impossible to debug, vendor lock-in
- Plugin architecture - didn't scale to mobile, security model became untenable
- Poor interoperability - couldn't easily integrate with JavaScript
- Maintenance burden for Adobe
