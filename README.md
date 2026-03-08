# 🤖 Luxonis-Oak-D-LR
These are python scripts for using a 🍇 Raspberry Pi to print object detection and x-y-z coordinates to terminal.

## Scripts
Tested on Rasbperry Pi 3 B+ with Python 3.3.0
- oak_pipeline.py (on the Pi) — talks to the camera, runs the AI model, and sends the data out over the network
- receiver.py (on Windows) — receives that data and prints it to your terminal
