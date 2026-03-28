# Jetson Nano Webcam Streaming + YOLO Demo Plan

## Goal

Build a simple real-time demo on Jetson Nano that:

1. receives video from a webcam connected to the Jetson
2. runs YOLO object detection with GPU acceleration
3. overlays bounding boxes and labels on the live video
4. streams the annotated output over the local network
5. can be started remotely over SSH with a small number of commands

This repository is the project workspace for that demo.

## Target Demo Scenario

The intended demo flow is:

1. Connect Jetson Nano and the developer laptop to the same network.
2. SSH into the Jetson from the laptop.
3. Start the demo service or run script.
4. Open the stream from another machine on the same network.
5. Confirm that object detection is running in real time on live webcam frames.

## Important Platform Constraints

This plan assumes the device is an actual Jetson Nano, not Xavier NX or Orin.

Jetson Nano has tighter CPU, GPU, memory, and software-version limits than newer Jetson devices. That changes the implementation strategy:

- Prefer TensorRT inference over general PyTorch runtime on device.
- Prefer smaller YOLO models first.
- Prefer RTSP for the first streaming target.
- Prefer a stable NVIDIA-supported stack over the newest Python package stack.

As of March 28, 2026, Jetson Nano should be treated as a JetPack 4 generation device. NVIDIA also states that the JetPack 4 codeline is end-of-life, and NVIDIA forum guidance states that JetPack 5.x does not support Jetson Nano. That matters because the project should be designed around Nano-compatible packages from the start instead of assuming the latest Jetson software stack.

## Recommended Architecture

### First Recommended Path

Use this as the main implementation path:

- Input: USB webcam via V4L2
- Inference: YOLO exported to ONNX, then converted to TensorRT
- Video pipeline: GStreamer / DeepStream-oriented pipeline
- Overlay: on-device bounding boxes and labels
- Output: RTSP stream over LAN
- Remote control: SSH

### Why This Path

This is the most practical route for a Jetson Nano demo because:

- TensorRT is the realistic path for GPU-accelerated inference on Nano.
- RTSP is much easier to make reliable than browser-first streaming.
- NVIDIA video tooling is better suited to Nano than a purely Python-first stack.
- A staged pipeline makes debugging easier: camera, then inference, then streaming.

### Avoid as the Primary Path

Do not treat this as the primary plan:

- latest Ultralytics package directly on Nano
- large YOLO models
- browser-only streaming as the first milestone
- training or heavy model conversion on Nano itself

These approaches are possible in some environments, but they add version risk and performance risk early.

## High-Level System Design

The end-to-end data flow should look like this:

1. Webcam frames are read from `/dev/video0`.
2. Frames are resized or normalized for the detector input.
3. TensorRT YOLO inference runs on the Jetson GPU.
4. Detection results are drawn onto frames.
5. The annotated video is encoded and served as an RTSP stream.
6. A laptop or another device opens the stream with VLC, `ffplay`, or another RTSP client.

## Development Strategy

Build this in layers instead of trying to finish the whole demo in one step.

### Phase 0: Lock the Base Environment

Before any application code, confirm the platform baseline:

- Jetson model
- JetPack version
- CUDA availability
- TensorRT availability
- available disk space
- available swap
- network reachability over SSH

Success criteria:

- SSH works reliably
- `tegrastats` runs
- CUDA/TensorRT packages are present
- webcam appears as a V4L2 device

### Phase 1: Camera Bring-Up

The first software milestone is not YOLO. It is a stable live camera feed.

Tasks:

- verify `/dev/video0` exists
- inspect supported resolutions and frame rates
- test raw preview or test capture
- decide the initial input resolution

Recommended initial resolutions:

- `640x480` for maximum stability
- `1280x720` only if the webcam and pipeline remain stable

Success criteria:

- webcam frames can be read continuously
- no intermittent disconnects
- stable FPS at the chosen input size

### Phase 2: YOLO Inference Only

Before adding streaming, get local detection working.

Tasks:

- choose a small YOLO model
- prepare the model off-device on a stronger machine
- export to ONNX
- build or load a TensorRT engine on the Jetson
- run inference on live frames
- draw boxes and labels

Recommended starting models:

- YOLOv5n
- YOLOv8n
- another nano-class detector if conversion is simpler

Do not start with medium or large models on Nano.

Success criteria:

- bounding boxes render correctly
- classes look reasonable
- pipeline runs continuously without crashes
- performance is stable enough for a live demo

### Phase 3: Add Streaming

Only after local inference works should streaming be added.

Tasks:

- encode the annotated output
- expose the stream over RTSP
- validate playback from another device on the same network
- measure end-to-end latency

Recommended first streaming target:

- RTSP viewed in VLC or `ffplay`

Reason:

- it is simple to validate
- it avoids browser-specific complexity
- it is sufficient for an internal demo

Success criteria:

- another machine can open the stream by IP and port
- stream is stable for several minutes
- boxes and labels are visible and synchronized with the live feed

### Phase 4: Performance Tuning

After the full path works, optimize for responsiveness.

Primary tuning levers:

- lower model size
- lower input resolution
- use FP16 TensorRT engine
- reduce inference frequency if needed
- tune encoder settings
- pin Jetson to max performance mode during the demo

Jetson Nano target should be defined as a stable demo, not a benchmark victory.

A realistic goal is a smooth and reliable low-latency demo, even if the FPS is modest.

### Phase 5: Demo Packaging

The final deliverable should be operationally simple.

Tasks:

- create one main run script
- create a setup script for dependencies and system tuning
- externalize camera and stream settings into config files
- optionally wrap the app in a `systemd` service
- document exact launch and view commands

Success criteria:

- user can SSH into the Jetson
- run one command
- open one RTSP URL
- see real-time detections

## Concrete Technical Recommendation

### Inference Path

Recommended order of preference:

1. DeepStream-compatible YOLO deployment
2. custom GStreamer pipeline + TensorRT inference wrapper
3. Python-first OpenCV inference only as a fallback prototype

The reason for this order is stability on Nano. The closer the runtime stays to NVIDIA's optimized path, the lower the risk during demo week.

### Streaming Path

Recommended order of preference:

1. RTSP
2. local display for debugging
3. WebRTC only after RTSP works well

RTSP is the right first target for a same-network demo because it is easier to stand up and easier to validate under SSH-driven workflows.

## Proposed Repository Layout

Use the repository as an operations and integration repo, not just a scratchpad.

```text
.
├── README.md
├── app/
│   ├── pipeline/
│   ├── inference/
│   ├── streaming/
│   └── main.py
├── configs/
│   ├── camera/
│   ├── model/
│   └── stream/
├── models/
│   ├── onnx/
│   └── tensorrt/
├── scripts/
│   ├── setup_jetson.sh
│   ├── run_demo.sh
│   ├── enable_max_perf.sh
│   └── check_camera.sh
├── docs/
│   ├── setup.md
│   ├── deployment.md
│   └── troubleshooting.md
└── samples/
```

### Directory Intent

- `app/`: runtime code for capture, inference, overlay, and stream control
- `configs/`: resolution, model path, labels, stream URL, and tuning parameters
- `models/`: exported model artifacts kept outside the main code logic
- `scripts/`: repeatable operational commands for setup and launch
- `docs/`: setup notes, device-specific issues, and troubleshooting

## Initial Implementation Roadmap

This is the recommended execution order for actual development.

### Milestone 1: Infrastructure Check

Deliverables:

- Jetson reachable by SSH
- camera detected
- baseline notes committed to the repo

### Milestone 2: Camera Validation Script

Deliverables:

- script that verifies webcam availability
- script that prints supported modes
- script or command sequence that captures a test frame

### Milestone 3: Local Inference Prototype

Deliverables:

- model artifact chosen
- inference path validated on live frames
- local overlay visible

### Milestone 4: Network Stream Prototype

Deliverables:

- RTSP endpoint exposed
- remote playback tested
- basic latency measurement noted

### Milestone 5: Demo Runner

Deliverables:

- `run_demo.sh`
- documented environment variables or config file
- simple startup instructions

### Milestone 6: Hardening

Deliverables:

- watchdog or restart strategy
- thermal and power notes
- fallback options if the camera or stream fails during the demo

## Performance and Reliability Checklist

Before calling the project demo-ready, verify:

- power supply is stable
- Jetson is in max performance mode for the demo
- swap is adequate if needed
- thermal throttling is not occurring
- camera reconnect behavior is known
- RTSP client can reconnect cleanly
- the chosen model can run continuously for at least 10 to 15 minutes

## Risk Register

### Risk 1: JetPack and Package Version Mismatch

Description:

Jetson Nano is not friendly to arbitrary current Python ML stacks.

Mitigation:

- lock versions early
- prefer Nano-supported NVIDIA tooling
- avoid rebuilding the stack in the last week

### Risk 2: YOLO Model Too Heavy for Real-Time Demo

Description:

A model may be accurate enough but too slow for a convincing live demo.

Mitigation:

- start with nano-class models
- reduce resolution early
- build the performance budget before polishing the UI

### Risk 3: Streaming Works Locally but Not Over Network

Description:

The pipeline can look good on the Jetson itself but fail due to network or client issues.

Mitigation:

- validate RTSP from a second machine as early as possible
- keep firewall and router assumptions simple
- document the exact client command used for validation

### Risk 4: Thermal or Power Instability

Description:

Jetson Nano demos can fail for operational reasons unrelated to the application logic.

Mitigation:

- use a known-good power supply
- test under continuous load
- monitor temperature and throttling before the demo day

## Suggested Definition of Done

The project should be considered complete when all of the following are true:

1. The Jetson is reachable over SSH on the local network.
2. A webcam connected to the Jetson is detected automatically.
3. The YOLO pipeline runs on GPU-accelerated inference.
4. The output stream includes bounding boxes and class labels.
5. Another device on the same network can view the live stream.
6. A single documented launch flow is enough to run the demo.
7. The system remains stable for a sustained live run.

## Practical Starting Point

The most pragmatic first implementation target is:

- USB webcam
- `640x480`
- small YOLO model
- TensorRT inference
- RTSP stream
- start from SSH with one script

This is the lowest-risk path to a demo that actually works on Jetson Nano.

## Next Repository Tasks

The next concrete tasks to implement in this repository are:

1. add `docs/setup.md` for Jetson environment preparation
2. add `scripts/check_camera.sh`
3. add `scripts/enable_max_perf.sh`
4. choose the first model export path
5. add `scripts/run_demo.sh`
6. add the first end-to-end inference pipeline stub

## References

- NVIDIA Jetson FAQ: https://developer.nvidia.com/embedded/faq
- NVIDIA forum note on JetPack 5.x and Jetson Nano: https://forums.developer.nvidia.com/t/jetpack-5-x-on-nano/274725
- DeepStream 6.0.1 Quickstart: https://docs.nvidia.com/metropolis/deepstream/6.0.1/dev-guide/text/DS_Quickstart.html
- NVIDIA JetCam: https://github.com/NVIDIA-AI-IOT/jetcam
- NVIDIA jetson-inference examples: https://github.com/dusty-nv/jetson-inference
