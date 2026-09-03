# yolov8s-pose

YOLOv8 pose estimation, 640x640. The smaller of the two pose models and the one to start with: 25.5 fps at 4 threads on an i7-12700H, against 10.8 for the medium.

**eRT `Model Type` = `1008`.** Output tensor is `[1, 56, 8400]` = 4 box + 1 class + 17 joints x 3.

## Originals

Downloaded from Ultralytics' published assets, release `v8.4.0`:

    https://github.com/ultralytics/assets/releases/download/v8.4.0/yolov8s-pose.pt

The `.onnx` is an export of that `.pt`, produced here - not a separate download.

| File | MB | SHA-256 (first 16) |
|------|----|--------------------|
| `yolov8s-pose.onnx` |  44.6 | `8e0aff5b16c20902` |
| `yolov8s-pose.pt` |  22.4 | `234314cd8baf6261` |

## Converted

| File | MB | Input | Layout | Output | SHA-256 (first 16) |
|------|----|-------|--------|--------|--------------------|
| `yolov8s-pose-dyn.tfl` |  11.5 | FLOAT32 `[1, 640, 640, 3]` | NHWC | FLOAT32 `[1, 56, 8400]` | `84784ad71a767a7f` |
| `yolov8s-pose-int8.tfl` |  11.5 | UINT8 `[1, 640, 640, 3]` | NHWC | UINT8 `[1, 56, 8400]` | `d94f8f4a5eeaf89f` |

See the [naming convention](../README.md#file-naming) for what each suffix
means. In short: **use `-dyn`**; `-int8` is published as a labelled specimen of
a failure mode, not as a usable model.

## Reproducing these

```bash
# 1. weights -> SavedModel. Run LOCALLY: this produces NHWC, which eRT needs.
#    The Ultralytics web UI emits NCHW, which eRT cannot load.
yolo export model=yolov8s-pose.pt format=saved_model imgsz=640

# 2. SavedModel -> .tfl
python3 export_tflite_int8.py yolov8s-pose_saved_model \
        -o yolov8s-pose-dyn.tfl --quant dynamic

# 3. verify before it reaches a device (no dependencies)
python3 tfl_inspect.py yolov8s-pose-dyn.tfl --camera 640x640x3
```

Scripts live in `ert-components/scripts/ai-utils/model-conversion/`.

## Licence

AGPL-3.0, as a derivative of Ultralytics YOLOv8. See [NOTICE](../NOTICE).
