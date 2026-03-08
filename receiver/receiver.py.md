receiver.py runs on the central computer and listens for incoming 
data from the Raspberry Pi over a TCP socket on port 5005. 
When a connection is established it continuously receives JSON 
payloads from the Pi, parses each one, and prints the timestamp, 
FPS, and all detected objects with their labels, confidence scores, 
and X/Y/Z spatial coordinates to the terminal.

```python
#!/usr/bin/env python3
"""
Central computer receiver -- receiver.py
Run with:  python receiver.py -- port 5005
"""

import argparse
import json
import socket

parser = argparse.ArgumentParser()
parser.add_argument("--port", type=int, default=5005)
args = parser.parse_args()

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(("0.0.0.0", args.port))
server.listen(1)
print(f"[Receiver] Listening on port {args.port}...")

conn, addr = server.accept()
print(f"[Receiver] Connected from {addr}")

buffer = ""
try:
    while True:
        chunk = conn.recv(4096).decode(errors="ignore")
        if not chunk:
            break
        buffer += chunk
        while "\n" in buffer:
            line, buffer = buffer.split("\n", 1)
            try:
                data = json.loads(line)
                fps  = data["fps"]
                dets = data["detections"]
                print(f"\n[t={data['timestamp']:.2f}] FPS: {fps}")
                for d in dets:
                    print(f"  {d['label']:15s} conf={d['confidence']:.2f} "
                          f"Z={d['z_mm']}mm  X={d['x_mm']}mm  Y={d['y_mm']}mm")
            except json.JSONDecodeError:
                pass
except KeyboardInterrupt:
    pass
finally:
    conn.close()
    server.close()
```
