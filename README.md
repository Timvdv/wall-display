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

**Record tap:**
```bash
sudo evemu-record /dev/input/event4 > tap.txt
```

**Replay tap:**
```bash
sudo evemu-play /dev/input/event4 < tap.txt
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
from fastapi import FastAPI, HTTPException
import subprocess

app = FastAPI()
EVENT_DEV = "/dev/input/event4"
TAP_FILE = "tap.txt"

@app.get("/wake")
async def wake():
    try:
        with open(TAP_FILE, "rb") as f:
            subprocess.run(["sudo", "evemu-play", EVENT_DEV], input=f.read(), check=True)
        return {"status": "Screen awakened"}
    except Exception as e:
        raise HTTPException(500, str(e))

@app.get("/brightness/{level}")
async def set_brightness(level: int):
    if 0 <= level <= 100:
        subprocess.run(["rpi-backlight", "-b", str(level)], check=True)
        return {"brightness": level}
    raise HTTPException(400, "Invalid brightness (0-100)")
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
