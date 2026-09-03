# hailo

Pre-compiled Hailo Executable Format binaries for the Hailo-8 accelerator.
Unlike everything else here, these were **not** produced by this repository's
conversion scripts — compiling a HEF needs Hailo's Dataflow Compiler.

| File               |   MB | Model            | eRT `Model Type` | SHA-256 (first 16) |
|--------------------|------|------------------|------------------|--------------------|
| `yolov8n.hef`      |  4.9 | YOLOv8n objdet   | `1006`           | `81d37b6c7b5eb961` |
| `yolov8s_pose.hef` | 10.6 | YOLOv8s pose     | `1008`           | `0ea260234250510c` |

`yolov8n.hef` has **NMS compiled in** — the file contains the string
`HAILO_NET_FLOW_YOLOV8_NMS`, so HailoRT returns a finished detection list and
eRT's model layer skips decoding entirely. `yolov8s_pose.hef` does not: it
publishes raw per-head tensors, which eRT decodes itself.

## Provenance — partially established, and worth reading before you rely on it

These files were found in an ert-components working tree, not produced here.
What has been checked:

**They are not verbatim copies of Hailo's published builds.** Both were compared
by SHA-256 against Hailo's compiled Model Zoo artefacts for Hailo-8:

    https://hailo-model-zoo.s3.eu-west-2.amazonaws.com/ModelZoo/Compiled/v<VER>/hailo8/<model>.hef

across `v2.11.0`, `v2.12.0`, `v2.13.0` and `v2.14.0`. **No version matched
either file.** That does not prove much on its own — HEF compilation is not
guaranteed reproducible across compiler versions and quantisation seeds, so a
local compile from the same source ONNX would also fail to match. It does rule
out a plain download of those four releases.

**The architecture is Hailo Model Zoo's.** `yolov8s_pose.hef` embeds the network
name `yolov8s_pose` and a head layout of `20x20x64, 20x20x1, 20x20x51, ...`,
which matches the Hailo Model Zoo config for that model exactly (51 = 17 COCO
joints x 3; 64 = DFL box head). The upstream ONNX that config names is:

    https://hailo-model-zoo.s3.eu-west-2.amazonaws.com/PoseEstimation/yolov8/yolov8s/pretrained/2023-06-11/yolov8s_pose.zip

**What is still unknown:** which compiler version produced these, and who ran
it. If that matters for your purpose, recompile from the upstream ONNX above
rather than trusting these.

## Licence

**AGPL-3.0** — the same as everything else here, and this catches people out.

The Hailo Model Zoo repository is MIT (`Copyright (c) Hailo, Inc.`), with no
carve-out for model files. But MIT covers what Hailo owns: their compiler,
tooling and configs. It cannot relicense weights Hailo did not author, and
Hailo's own config for `yolov8s_pose` records `framework: pytorch`,
`training_data: coco keypoints train2017` and layer names in Ultralytics'
YOLOv8 module form (`/model.22/cv2.2/cv2.2.2/Conv`).

Compiling weights to a new format is a transformation, not fresh authorship, so
a YOLOv8 HEF remains an AGPL-3.0 derivative of Ultralytics YOLOv8. See
[NOTICE](../NOTICE).
