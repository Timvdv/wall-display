# Raspberry Pi Touchscreen Wake & Brightness Control with FastAPI and PM2

This repository documents how to:

1. Prevent the Raspberry Pi touchscreen from going to sleep too quickly.  
2. Simulate a tap to wake the screen programmatically.  
3. Adjust screen brightness via the command line.  
4. Expose these controls through a FastAPI server.  
5. Automatically start the FastAPI server at boot using PM2.  
6. (Optional) Check out [my YouTube channel](https://www.youtube.com/@timvdv) for more information.

---

## Table of Contents

1. [Overview](#overview)  
2. [Disable or Adjust Console Blanking](#disable-or-adjust-console-blanking)  
3. [Simulating a Screen Tap](#simulating-a-screen-tap)  
   - [Identify the Touchscreen Event Device](#identify-the-touchscreen-event-device)  
   - [Record a Tap Event](#record-a-tap-event)  
   - [Replay the Tap Event](#replay-the-tap-event)  
4. [Adjusting Brightness](#adjusting-brightness)  
5. [FastAPI Server](#fastapi-server)  
   - [Install Dependencies](#install-dependencies)  
   - [Create `main.py` for FastAPI](#create-mainpy-for-fastapi)  
   - [Run FastAPI Manually](#run-fastapi-manually)  
6. [Manage FastAPI with PM2](#manage-fastapi-with-pm2)  
   - [Option A: Direct Python Interpreter](#option-a-direct-python-interpreter)  
   - [Option B: Shell Script Wrapper](#option-b-shell-script-wrapper)  
7. [Notes](#notes)

---

## Overview

By default, the Raspberry Pi may blank its screen after a period of inactivity. When using the official Raspberry Pi touchscreen (connected via the DSI/display connector), simply sending `xset` or DPMS commands might not wake it. Instead, you may need to:

- Adjust the kernel’s console blanking parameter (`consoleblank`) to prevent or delay blanking.  
- Use `evemu` tools to simulate touch events, effectively “tapping” the screen to wake it.  
- Use `rpi-backlight` to control brightness.  
- Expose these features via a FastAPI app, which can be called by other services (e.g., home automation).  
- Ensure the FastAPI app starts automatically on boot with PM2.  
- Check out more details on [my YouTube channel](https://www.youtube.com/@timvdv).

---

## Disable or Adjust Console Blanking

The Raspberry Pi (particularly in newer distributions) often stores its kernel boot parameters in `/boot/firmware/cmdline.txt`. Look for a parameter like `consoleblank=600` (which means 600 seconds = 10 minutes). To change or remove it:

1. **Check the contents**:  
   ```bash
   cat /boot/firmware/cmdline.txt
