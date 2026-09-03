# yolov8n

YOLOv8 object detection, 640x640. The smallest COCO detector here, and the quickest way to prove a pipeline end to end.

**eRT `Model Type` = `1006`.** Output tensor is `[1, 84, 8400]` = 4 box + 80 COCO classes.

## Originals

Downloaded from Ultralytics' published assets, release `v8.4.0`:

    https://github.com/ultralytics/assets/releases/download/v8.4.0/yolov8n.pt

The `.onnx` is an export of that `.pt`, produced here - not a separate download.

| File | MB | SHA-256 (first 16) |
|------|----|--------------------|
| `yolov8n.onnx` |  12.2 | `439701b74e6ffdbb` |
| `yolov8n.pt` |   6.2 | `f59b3d833e2ff32e` |

## Converted

| File | MB | Input | Layout | Output | SHA-256 (first 16) |
|------|----|-------|--------|--------|--------------------|
| `yolo8n.tfl` |   3.2 | FLOAT32 `[1, 640, 640, 3]` | NHWC | FLOAT32 `[1, 84, 8400]` | `8efdbba126f7f313` |

See the [naming convention](../README.md#file-naming) for what each suffix
means. In short: **use `-dyn`**; `-int8` is published as a labelled specimen of
a failure mode, not as a usable model.

## Reproducing these

```bash
# 1. weights -> SavedModel. Run LOCALLY: this produces NHWC, which eRT needs.
#    The Ultralytics web UI emits NCHW, which eRT cannot load.
yolo export model=yolov8n.pt format=saved_model imgsz=640

# 2. SavedModel -> .tfl
python3 export_tflite_int8.py yolov8n_saved_model \
        -o yolov8n-dyn.tfl --quant dynamic

# 3. verify before it reaches a device (no dependencies)
python3 tfl_inspect.py yolov8n-dyn.tfl --camera 640x640x3
```

Scripts live in `ert-components/scripts/ai-utils/model-conversion/`.

## Licence

AGPL-3.0, as a derivative of Ultralytics YOLOv8. See [NOTICE](../NOTICE).
