# Luxonis-Oak-D-LR
Python scripts for using a raspberry pi to print object detection and x-y-z coordinates to terminal 
These scripts are made using rasbperry pi 3 b+ with python 3.3.0 installed
Two scripts:
  oak_pipeline.py (on the Pi) — talks to the camera, runs the AI model, and sends the data out over the network
  receiver.py (on Windows) — receives that data and prints it to your terminal
