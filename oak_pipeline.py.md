oak_pipeline.py runs on the Raspberry Pi and controls the OAK-D LR camera. 
It creates a stereo depth pipeline using the left and right monochrome cameras, 
feeds the RGB camera into a YOLOv6-nano object detection model, and combines the 
two to calculate the real-world 3D position of each detected object. 
On every frame it packages the results into a JSON payload containing a timestamp, 
current FPS, and a list of detected objects — each with a label, confidence score, 
and X/Y/Z coordinates in millimeters — and sends that over a TCP socket to the central 
computer.

```python
#!/usr/bin/env python3
"""
OAK-D LR — Spatial Object Detection + FPS sender
Run inside (oak_env) on the Raspberry Pi:
    python3 oak_pipeline.py --host 192.168.x.x --port 5005
"""

import argparse
import json
import socket
import time
import depthai as dai

# ─── Args ────────────────────────────────────────────────────────────────────
parser = argparse.ArgumentParser()
parser.add_argument("--host", type=str, required=True,
                    help="IP address of the central computer")
parser.add_argument("--port", type=int, default=5005)
args = parser.parse_args()

# ─── FPS tracker ─────────────────────────────────────────────────────────────
class FPSCounter:
    def __init__(self, window=30):
        self.timestamps = []
        self.window = window

    def tick(self):
        now = time.monotonic()
        self.timestamps.append(now)
        if len(self.timestamps) > self.window:
            self.timestamps.pop(0)

    def fps(self):
        if len(self.timestamps) < 2:
            return 0.0
        elapsed = self.timestamps[-1] - self.timestamps[0]
        return (len(self.timestamps) - 1) / elapsed if elapsed > 0 else 0.0

# ─── Sender node ─────────────────────────────────────────────────────────────
class DetectionSender(dai.node.HostNode):
    """Receives spatial detections, computes FPS, sends JSON to central PC."""

    def __init__(self):
        dai.node.HostNode.__init__(self)
        self.sendProcessingToPipeline(True)
        self.fps = FPSCounter()
        self.sock = None
        self._connect()

    def _connect(self):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.connect((args.host, args.port))
        print(f"[Sender] Connected to {args.host}:{args.port}")

    def build(self, detections: dai.Node.Output):
        self.link_args(detections)
        return self

    def process(self, detections):
        self.fps.tick()
        current_fps = round(self.fps.fps(), 2)

        objects = []
        for det in detections.detections:
            objects.append({
                "label":      det.labelName,
                "confidence": round(det.confidence, 3),
                "x_mm":       int(det.spatialCoordinates.x),
                "y_mm":       int(det.spatialCoordinates.y),
                "z_mm":       int(det.spatialCoordinates.z),   # Z = forward range
                "bbox": {
                    "xmin": round(det.xmin, 3),
                    "ymin": round(det.ymin, 3),
                    "xmax": round(det.xmax, 3),
                    "ymax": round(det.ymax, 3),
                }
            })

        payload = {
            "timestamp": time.time(),
            "fps":       current_fps,
            "detections": objects,
        }

        try:
            msg = (json.dumps(payload) + "\n").encode()
            self.sock.sendall(msg)
        except (BrokenPipeError, ConnectionResetError):
            print("[Sender] Connection lost, reconnecting...")
            self._connect()

# ─── Pipeline ────────────────────────────────────────────────────────────────
MODEL = dai.NNModelDescription("yolov6-nano")   # downloads automatically
SIZE  = (640, 400)
FPS   = 10

with dai.Pipeline() as pipeline:
    # Camera nodes — OAK-D LR has CAM_A (RGB), CAM_B (left mono), CAM_C (right mono)
    cam_rgb   = pipeline.create(dai.node.Camera).build(
                    dai.CameraBoardSocket.CAM_A, sensorFps=FPS)
    cam_left  = pipeline.create(dai.node.Camera).build(
                    dai.CameraBoardSocket.CAM_B, sensorFps=FPS)
    cam_right = pipeline.create(dai.node.Camera).build(
                    dai.CameraBoardSocket.CAM_C, sensorFps=FPS)

    # Stereo depth
    stereo = pipeline.create(dai.node.StereoDepth)
    stereo.setExtendedDisparity(True)
    stereo.setOutputSize(SIZE[0], SIZE[1])
    cam_left.requestOutput(SIZE).link(stereo.left)
    cam_right.requestOutput(SIZE).link(stereo.right)

    # Spatial detection network
    sdn = pipeline.create(dai.node.SpatialDetectionNetwork).build(
        cam_rgb, stereo, MODEL
    )
    sdn.input.setBlocking(False)
    sdn.setBoundingBoxScaleFactor(0.5)
    sdn.setDepthLowerThreshold(200)    # mm — ignore closer than 20 cm
    sdn.setDepthUpperThreshold(10000)  # mm — ignore farther than 10 m

    # Sender host node
    sender = pipeline.create(DetectionSender).build(sdn.out)

    print(f"[Pipeline] Running. Streaming to {args.host}:{args.port}")
    pipeline.run()
```
