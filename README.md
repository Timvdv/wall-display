# Raspberry Pi Touchscreen Wake & Brightness Control with FastAPI, PM2, and Kiosk Mode

## Table of Contents
1. [Overview](#1-overview)
2. [Disable Console Blanking](#2-disable-console-blanking)
3. [Simulate Screen Tap](#3-simulate-screen-tap)
4. [Adjust Brightness](#4-adjust-brightness)
5. [FastAPI Server](#5-fastapi-server)
6. [PM2 Management](#6-pm2-management)
7. [Chromium Kiosk](#7-chromium-kiosk)
8. [Desktop Icon](#8-desktop-icon)
9. [Notes](#9-notes)

---

### 1. Overview <a name="1-overview"></a>
Control Raspberry Pi touchscreen wake behavior and brightness through automated scripts and web APIs.

---

### 2. Disable Console Blanking <a name="2-disable-console-blanking"></a>
Edit kernel parameters:
```bash
sudo nano /boot/firmware/cmdline.txt
# I've added at the end, so the screen automatically goes to sleep `consoleblank=600`
sudo reboot
```

---

### 3. Simulate Screen Tap <a name="3-simulate-screen-tap"></a>
**Identify device:**
```bash
grep -i touch /proc/bus/input/devices
```

You can dive deeper with this command, in my case `Goodix Capacitive TouchScreen` was the name of the touch screen i'm using listed above
```bash
grep -i -A 5 "Goodix Capacitive TouchScreen" /proc/bus/input/devices
```

**Record tap:**
Make sure to set the `event4` to the event you see in the previous command

```bash
sudo evemu-record /dev/input/event4 > tap_event.txt
```

**Replay tap:**
```bash
sudo evemu-play /dev/input/event4 < tap_event.txt
```

---

### 4. Adjust Brightness <a name="4-adjust-brightness"></a>
Install control tool:
```bash
sudo pip3 install rpi-backlight
```

Set brightness:
```bash
rpi-backlight -b 75  # 75% brightness
```

---

### 5. FastAPI Server <a name="5-fastapi-server"></a>
**Install dependencies:**
```bash
sudo pip3 install fastapi uvicorn
```

**main.py:**
```python
import subprocess
from fastapi import FastAPI, HTTPException, BackgroundTasks
import os
import logging

app = FastAPI()

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Use an absolute path for TAP_EVENT_FILE
TAP_EVENT_FILE = os.path.join(os.path.dirname(__file__), "tap_event.txt")

def run_evemu_play():
    """
    Background task to run evemu-play and send an Enter keypress.
    """
    try:
        logger.info("Background task: Starting evemu-play subprocess.")

        # Run the subprocess and send input using subprocess.run
        result = subprocess.run(
            ["sudo", "evemu-play", TAP_EVENT_FILE],
            input=b'\n',  # Send the Enter keypress
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            timeout=5  # Adjust the timeout as needed
        )

        if result.returncode != 0:
            error_message = result.stderr.decode().strip()
            logger.error(f"Background task: evemu-play failed with return code {result.returncode}: {error_message}")
        else:
            logger.info("Background task: evemu-play executed successfully.")

    except subprocess.TimeoutExpired:
        logger.error("Background task: evemu-play subprocess timed out.")
    except Exception as e:
        logger.error(f"Background task: Unexpected error: {str(e)}")

@app.get("/wake")
async def wake_screen(background_tasks: BackgroundTasks):
    """
    Endpoint to wake the screen by simulating a tap.
    """
    logger.info("API: Received request to wake the screen.")

    # Add the background task
    background_tasks.add_task(run_evemu_play)

    # Return success immediately
    return {"status": "Screen wake command initiated."}

@app.get("/brightness/{level}")
async def set_brightness(level: int):
    """
    Endpoint to set the screen brightness.
    """
    if level < 0 or level > 100:
        logger.warning(f"API: Invalid brightness level requested: {level}")
        raise HTTPException(status_code=400, detail="Brightness must be between 0 and 100.")

    try:
        logger.info(f"API: Setting brightness to {level}.")
        command = ["rpi-backlight", "-b", str(level)]

        # Run the subprocess without waiting for it to complete
        result = subprocess.run(
            command,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            timeout=5  # Adjust the timeout as needed
        )

        if result.returncode != 0:
            error_message = result.stderr.decode().strip()
            logger.error(f"API: rpi-backlight failed with return code {result.returncode}: {error_message}")
            raise HTTPException(status_code=500, detail=error_message)

        logger.info(f"API: Brightness set to {level} successfully.")
        return {
            "status": "Brightness set",
            "level": level,
            "output": result.stdout.decode().strip()
        }

    except subprocess.TimeoutExpired:
        logger.error("API: rpi-backlight subprocess timed out.")
        raise HTTPException(status_code=500, detail="Brightness adjustment timed out.")
    except Exception as e:
        logger.error(f"API: Unexpected error: {str(e)}")
        raise HTTPException(status_code=500, detail="An unexpected error occurred.")
```

**Sudoers config:**
```bash
# Run sudo visudo and add:
www-data ALL=(ALL) NOPASSWD: /usr/bin/evemu-play, /usr/bin/rpi-backlight
```

---

### 6. PM2 Management <a name="6-pm2-management"></a>
**Install PM2:**
```bash
curl -sL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install nodejs
sudo npm install -g pm2
```

**Start service:**
```bash
pm2 start "uvicorn main:app --host 0.0.0.0 --port 8000" --name screen-api
pm2 save
pm2 startup
# Run generated command
```

---

### 7. Chromium Kiosk <a name="7-chromium-kiosk"></a>
**Minimal X setup:**
```bash
sudo apt install xserver-xorg xinit matchbox-window-manager chromium-browser
```

**startx.sh:**
```bash
#!/bin/bash
xset -dpms
xset s off
matchbox-window-manager -use_cursor no &
chromium-browser --kiosk --incognito http://your-dashboard-url
```

**Systemd service (/etc/systemd/system/kiosk.service):**
```ini
[Unit]
Description=Kiosk Mode
After=network.target

[Service]
User=pi
ExecStart=/usr/bin/startx /home/pi/startx.sh -- -nocursor
Restart=always

[Install]
WantedBy=multi-user.target
```

---

### 8. Desktop Icon <a name="8-desktop-icon"></a>
Create `~/Desktop/Kiosk.desktop`:
```ini
[Desktop Entry]
Name=Launch Kiosk
Exec=chromium-browser --kiosk http://your-url
Icon=chromium
Type=Application
```

---

### 9. Notes <a name="9-notes"></a>
- Test touch events with `evtest`
- Use udev rules for consistent event device paths
- Consider firewall rules for FastAPI port (8000)
- Monitor PM2 logs: `pm2 logs screen-api`

[Full YouTube Guide](https://youtube.com/@timvdv)
