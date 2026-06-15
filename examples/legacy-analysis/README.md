# Legacy Analysis: Understanding the Flash App

This directory contains analysis materials for the original `facerecognitioninbrowser` Flash application.

## Files Analyzed

| File | Purpose |
|------|---------|
| FaceRecognizerForBrowserUsingCamera.html | XHTML entry point |
| FaceRecognizerForBrowserUsingCamera.swf | Compiled Flex application |
| swfobject.js | Flash embedding library |
| framework_*.swf | Flex 4.6 framework runtime |
| spark_*.swf | Spark UI components |
| mx_*.swf | MX layout components |
| rpc_*.swf | RPC/networking |
| faces/trainingSet/images/*.jpg | 23 training images (3 subjects) |
| faces/trainingSet/trainedData.php | 493KB serialized model |
| faces/trainingSet/upload.php | PHP upload handler |
| faces/trainingSet/getTrainingSetData.php | PHP metadata provider |

## Key Observations

1. **Total Flash footprint**: ~2.4 MB for the Flex framework SWFs alone
2. **Training data**: 23 images for 3 people, ~750 KB JPEGs
3. **Serialized model**: 493 KB proprietary format
4. **Backend**: Basic PHP scripts (< 200 lines total)
5. **Browser history**: Custom history management for Flash navigation
