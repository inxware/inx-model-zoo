# inx-model-zoo

Vision models converted for the inxware embedded runtime (eRT), together with
the originals they were converted from, so any conversion here can be repeated
rather than trusted.

> **Licensing: these are AGPL-3.0 derivatives.** Every model here descends from
> Ultralytics YOLOv8 weights. Read [NOTICE](NOTICE) before shipping any of them
> in a product — the AGPL's network clause reaches software served over a
> network, not only software distributed as a binary.

Model files are held in **Git LFS** (see `.gitattributes`). `yolov8m-pose.onnx`
is 101 MB, over GitHub's 100 MB per-file hard limit, so LFS is required here
rather than merely tidy. Clone with `git lfs install` already done, or the
working tree fills with pointer files.

## Using one in eRT

The `ml_image_inference` block needs its **Model Type** parameter to match the
model — there is no auto-detect, and a mismatch fails the load rather than
falling back:

| Model kind               | Model Type | Notes                                     |
|--------------------------|------------|-------------------------------------------|
| YOLOv8 object detection  | `1006`     |                                           |
| YOLOv8 pose estimation   | `1008`     |                                           |
| YOLOv5 object detection  | `1003`     |                                           |

`0` is **not** a "don't care" — it fails with `EHS_ML_MODEL_TYPE_ERR` (12).

## Inventory

`layout` is the one that silently costs you a day: eRT requires **NHWC**. An
Ultralytics *web UI* export produces NCHW, which eRT cannot consume.

### Pose estimation — output `[1, 56, 8400]` = 4 box + 1 class + 17 joints x 3

| File                                     |   MB | In      | Layout | Out     | Status                          |
|------------------------------------------|------|---------|--------|---------|---------------------------------|
| `yolov8s-pose/tflite/yolov8s-pose-dyn.tfl`  | 11.5 | FLOAT32 | NHWC   | FLOAT32 | **recommended**                 |
| `yolov8m-pose/tflite/yolov8m-pose-dyn.tfl`  | 25.7 | FLOAT32 | NHWC   | FLOAT32 | **recommended**, more accurate  |
| `yolov8s-pose/tflite/yolov8s-pose-int8.tfl` | 11.5 | UINT8   | NHWC   | UINT8   | **broken — do not use**, see below |
| `hailo/yolov8s_pose.hef`                    | 10.6 | —       | —      | —       | Hailo-8; provenance unconfirmed |

### Object detection — output `[1, 84, 8400]` = 4 box + 80 classes

| File                            |   MB | In      | Layout   | Out     | Status                       |
|---------------------------------|------|---------|----------|---------|------------------------------|
| `yolov8n/tflite/yolo8n.tfl`     |  3.2 | FLOAT32 | NHWC     | FLOAT32 | good                         |
| `yolov8m/tflite/yolo8md.tfl`    | 25.1 | FLOAT32 | NHWC     | FLOAT32 | good                         |
| `yolov8m/tflite/yolo8mu8.tfl`   | 25.2 | UINT8   | NHWC     | FLOAT32 | uint8 input, float output    |
| `yolov8m/tflite/yolo8m8.tfl`    | 25.3 | FLOAT32 | **NCHW** | FLOAT32 | **eRT cannot load this**     |
| `yolov8m/tflite/yolov8m32.tfl`  | 99.0 | FLOAT32 | **NCHW** | FLOAT32 | **eRT cannot load this**     |
| `hailo/yolov8n.hef`             |  5.1 | —       | —        | —       | NMS compiled in; provenance unconfirmed |

The two NCHW files are kept because they are what a web-UI export looks like,
and recognising one is most of diagnosing it — not because they are usable.

The `.hef` files were found in an ert-components working tree rather than
produced here; whether they came from the Hailo model zoo or were compiled
locally is **not established**. Confirm before relying on their provenance.

## Why there is no full-int8 pose model that works

Full int8 quantisation is measurably faster, and it destroys this model. The
YOLOv8 head is one concatenated tensor carrying values on two incompatible
scales — box and keypoint coordinates spanning 0-640, and class scores and
keypoint visibilities spanning 0-1 — and TFLite gives an activation tensor a
single per-tensor scale. Measured on `yolov8s-pose-int8.tfl`:

    output scale = 2.789376, zero point = 14

A confidence of 0.90 therefore encodes to code 14, which *is* the zero point:
every score and every joint visibility quantises to zero. The full 0-1
probability range spans **0.36 of one code** out of 255.

This is not a calibration problem and more calibration images do not help — it
is the arithmetic of one scale over two ranges. Dynamic-range quantisation
(int8 weights, float32 activations and IO) is the correct trade and is what the
`-dyn` files are. Per-channel quantisation applies to weights, not activations,
so it does not rescue this either.

## Measured throughput

640x640, TFLite + XNNPACK, i7-12700H (6 P-cores + 8 E-cores, 20 threads,
AVX2 + AVX-VNNI), 10 iterations after 3 warm-up:

| Model            | 1 thread | 4 threads | 8 threads | 20 threads |
|------------------|----------|-----------|-----------|------------|
| yolov8s-pose dyn | 10.4 fps | 25.5 fps  | 26.0 fps  | 1.1 fps    |
| yolov8m-pose dyn |  4.3 fps | 10.8 fps  | 11.2 fps  | 0.8 fps    |

**Do not set Thread Number to the core count.** At 20 threads this CPU is 24x
slower than at 8: the P-core/E-core split plus hyperthreading makes
oversubscription catastrophic rather than merely unhelpful. 4 threads reaches
98% of the best result, which is why eRT's auto mode caps there
(`EHS_ML_TFLITE_AUTO_THREADS_MAX`).

## Regenerating any of this

Conversion scripts, and the reasoning behind each converter setting, are in
`ert-components/scripts/ai-utils/model-conversion/`:

```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
pip install ultralytics          # AGPL-3.0, tool only - see NOTICE

# .pt -> SavedModel. Do this LOCALLY: it is NHWC, which eRT needs.
yolo export model=yolov8s-pose.pt format=saved_model imgsz=640

# SavedModel -> .tfl, int8 weights with a float32 boundary
python3 export_tflite_int8.py yolov8s-pose_saved_model \
        -o yolov8s-pose-dyn.tfl --quant dynamic

# verify before it goes near a device (no dependencies)
python3 tfl_inspect.py yolov8s-pose-dyn.tfl --camera 640x640x3
```

`tfl_inspect.py` is a zero-dependency FlatBuffer reader — it is the quickest way
to catch an NCHW export or an unexpected tensor dtype before a device does.
