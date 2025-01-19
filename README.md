# Raspberry Pi Touchscreen Wake & Brightness Control with FastAPI and PM2

This repository documents how to:

1. Prevent the Raspberry Pi touchscreen from going to sleep too quickly.  
2. Simulate a tap to wake the screen programmatically.  
3. Adjust screen brightness via the command line.  
4. Expose these controls through a FastAPI server.  
5. Automatically start the FastAPI server at boot using PM2.  
6. Check out [my YouTube channel](https://www.youtube.com/@timvdv) for more information.

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

By default, the Raspberry Pi may blank its screen after a period of inactivity. When using the official Raspberry Pi touchscreen (connected via the DSI/display connector), simply sending xset or DPMS commands might not wake it. Instead, you may need to:

- Adjust the kernel’s console blanking parameter (`consoleblank`) to prevent or delay blanking.  
- Use evemu tools to simulate touch events, effectively “tapping” the screen to wake it.  
- Use rpi-backlight to control brightness.  
- Expose these features via a FastAPI app, which can be called by other services (e.g., home automation).  
- Ensure the FastAPI app starts automatically on boot with PM2.  
- Check out more details on [my YouTube channel](https://www.youtube.com/@timvdv).

---

## Disable or Adjust Console Blanking

The Raspberry Pi (particularly in newer distributions) often stores its kernel boot parameters in `/boot/firmware/cmdline.txt`. Look for a parameter like `consoleblank=600` (which means 600 seconds = 10 minutes). To change or remove it:

1. **Check the contents**:
    
    cat /boot/firmware/cmdline.txt

2. **Edit the file**:
    
    sudo nano /boot/firmware/cmdline.txt

3. **Remove or adjust** `consoleblank=600`. For instance, set it to `0` to disable blanking or increase/decrease the timeout:
    
    consoleblank=0

4. **Reboot** for the change to take effect:
    
    sudo reboot

---

## Simulating a Screen Tap

If you need to forcibly wake the screen when it’s blanked, the best approach is to simulate an actual touch event.

### Identify the Touchscreen Event Device

1. Run:
    
    grep -i touch /proc/bus/input/devices

2. Look for a line like:

    N: Name="10-005d Goodix Capacitive TouchScreen"
    H: Handlers=mouse0 event4
    
   The `event4` (or similar) is your device file: `/dev/input/event4`.

### Record a Tap Event

1. Install `evemu-tools` if needed:
    
    sudo apt-get update
    sudo apt-get install evemu-tools

2. Record a tap:
    
    sudo evemu-record /dev/input/event4 > tap_event.txt

3. When prompted, physically tap the screen. Press Ctrl+C to stop.

### Replay the Tap Event

To simulate that tap:

    sudo evemu-play /dev/input/event4 < tap_event.txt

This command replays the touch event from `tap_event.txt`, effectively “waking” the screen.

---

## Adjusting Brightness

Use [rpi-backlight](https://github.com/linusg/rpi-backlight) to adjust the brightness of the official Raspberry Pi touchscreen:

1. **Install** (if not already).

   **Note**: We installed pip packages *globally* using `sudo pip3 install`, which is a bit of a workaround but helps ensure all scripts and services can access the Python packages system-wide:
   
       sudo pip3 install rpi-backlight

2. **Set Brightness** (0 to 100):
   
       rpi-backlight -b 50
   
   This sets the backlight brightness to 50%.

---

## FastAPI Server

### Install Dependencies

1. **FastAPI and Uvicorn**:
   
       sudo pip3 install fastapi uvicorn

2. **(Optional) rpi-backlight** if not already installed:
   
       sudo pip3 install rpi-backlight

3. **Set up `evemu-tools`** as described above for tap simulation.

### Create `main.py` for FastAPI

Below is a minimal FastAPI app with two GET endpoints:

- `GET /wake` to replay the recorded tap event.  
- `GET /brightness/{level}` to set the brightness level (0–100).

    from fastapi import FastAPI, HTTPException
    import subprocess

    app = FastAPI()

    # Adjust the paths to match your device and file locations:
    EVENT_DEVICE = "/dev/input/event4"
    TAP_EVENT_FILE = "tap_event.txt"  # Ensure this is the correct path

    @app.get("/wake")
    async def wake_screen():
        try:
            with open(TAP_EVENT_FILE, "rb") as f:
                tap_data = f.read()
            subprocess.run(
                ["sudo", "evemu-play", EVENT_DEVICE],
                input=tap_data,
                check=True
            )
            return {"status": "Screen woken up"}
        except Exception as e:
            raise HTTPException(status_code=500, detail=str(e))

    @app.get("/brightness/{level}")
    async def set_brightness(level: int):
        if not (0 <= level <= 100):
            raise HTTPException(status_code=400, detail="Brightness must be between 0 and 100")
        try:
            subprocess.run(["rpi-backlight", "-b", str(level)], check=True)
            return {"status": "Brightness set", "level": level}
        except Exception as e:
            raise HTTPException(status_code=500, detail=str(e)}

> Note: The `sudo` usage in `evemu-play` may require passwordless sudo privileges.  
> Update your `sudoers` file (via `sudo visudo`) with lines like:
> 
>     pi ALL=(ALL) NOPASSWD: /usr/bin/evemu-play, /usr/bin/rpi-backlight
> 
> Adjust username and paths as necessary.

### Run FastAPI Manually

1. Navigate to the directory with `main.py`:
   
       cd /path/to/your/app

2. Start the server:
   
       uvicorn main:app --host 0.0.0.0 --port 8000

3. Test endpoints:
   
   - **Wake Screen**:  
         
         GET http://<raspberry_pi_ip>:8000/wake

   - **Set Brightness**:  
         
         GET http://<raspberry_pi_ip>:8000/brightness/50

---

## Manage FastAPI with PM2

[PM2](https://pm2.keymetrics.io/) is a Node.js process manager, but it can manage any script or process. We’ll use it to start our Uvicorn server on boot.

1. **Install Node.js & PM2**:
   
       curl -sL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
       sudo apt-get install -y nodejs
       sudo npm install -g pm2

2. **Verify Installation**:
   
       pm2 --version

### Option A: Direct Python Interpreter

    cd /path/to/your/app
    pm2 start python3 --name fastapi-server -- -m uvicorn main:app --host 0.0.0.0 --port 8000

### Option B: Shell Script Wrapper

1. **Create `start_uvicorn.sh`** in your app folder:

       nano start_uvicorn.sh

   **Contents**:

       #!/bin/bash
       uvicorn main:app --host 0.0.0.0 --port 8000

2. **Make it executable**:

       chmod +x start_uvicorn.sh

3. **Start with PM2**:

       pm2 start ./start_uvicorn.sh --name fastapi-server

### Configure PM2 for Startup

Once your app is running under PM2:

1. **Save the process list**:

       pm2 save

2. **Generate startup script**:

       pm2 startup systemd

   Copy and run the command PM2 prints out (it typically has `sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u <user>`).

3. **Reboot** and check:

       sudo reboot
       pm2 list

   The `fastapi-server` process should be running.

---

## Notes

- **Permissions**: Interacting with `/dev/input/eventX` and backlight often requires root privileges. If you don’t want to run the entire FastAPI app as `root`, configure passwordless `sudo` for the specific commands.  
- **File Paths**: Make sure your recorded tap event (`tap_event.txt`) is located where your FastAPI app expects it.  
- **Auto-rotating Event Numbers**: Sometimes event numbers can change after reboots. If `/dev/input/event4` changes to another number, you’ll need to update your code or use udev rules to assign a consistent symlink.  
- **RESTful Note**: We used **GET** for convenience, but typically changing resources (like brightness) is done with **POST** or **PUT** in RESTful APIs.  
- **Global pip Installs**: Installing packages with `sudo pip3 install` is generally not recommended for all setups, but it can simplify reproducing the environment. Doing so gives system-wide access to libraries needed by your scripts and PM2-managed processes.

---

**For more details, tutorials, and projects, check out my [YouTube channel](https://www.youtube.com/@timvdv).**
