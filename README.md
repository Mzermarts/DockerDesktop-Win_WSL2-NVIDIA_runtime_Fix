# DockerDesktop-Win_WSL2-NVIDIA_runtime_Fix
"Fix for Docker Desktop 4.79.0 stuck on 'Starting the Docker Engine…' due to NVIDIA Container Toolkit conflicts in WSL2. No data loss, no downgrade needed."
# 🚀 Docker Desktop 4.79.0 Fix: NVIDIA Container Toolkit Conflict in WSL2

![Docker](https://img.shields.io/badge/Docker-24.0.7-blue)
![WSL2](https://img.shields.io/badge/WSL2-Ubuntu%2022.04-orange)
![NVIDIA](https://img.shields.io/badge/NVIDIA-Container%20Toolkit-green)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## 📌 Problem
Docker Desktop **4.79.0** (Windows 11 + WSL2) gets stuck on **"Starting the Docker Engine…"** with no error logs.
**Root cause**: Conflict with **NVIDIA Container Toolkit** (`nvidia-container-runtime`) in WSL2, causing a startup loop.

**Symptoms**:
- Docker Desktop hangs indefinitely on startup.
- No error messages in logs (`service.txt`).
- WSL2 distros (`docker-desktop`, `docker-desktop-data`) may be in a `Stopped` state.

---

## ✅ Solution (Tested & Working)
**No data loss   No downgrade | 10 minutes max**

### 1️⃣ Remove NVIDIA Container Toolkit from WSL (Ubuntu-22.04)
```bash
sudo apt remove --purge nvidia-container-toolkit libnvidia-container-tools libnvidia-container1 nvidia-container-toolkit-base
sudo apt autoremove
---
2️⃣ Clean daemon.json in WSL
# Check current config
sudo cat /etc/docker/daemon.json

# Replace with empty config (or delete the file)
echo "{}" | sudo tee /etc/docker/daemon.json
---
3️⃣ Clean daemon.json in Docker Desktop (Windows)

Open Docker Desktop → Settings → Docker Engine.
Remove all nvidia runtime references:

{
  "default-runtime": "runc",
  "log-driver": "json-file",
  "log-opts": {
    "max-file": "7",
    "max-size": "100m"
  }
}

Click Apply & Restart.
or, edit json file with notepad and remove informations about the runtime ...
save ... note pad step is better, with docker desktop stopped, verify all processus and services of docker are stop, killed !
---
4️⃣ Restart Docker Desktop


Quit Docker Desktop from the system tray.
Wait 2-3 minutes for all processes to stop.
Relaunch Docker Desktop.

🎯 Results
✅ Docker Desktop starts without errors.
✅ No data loss (containers, volumes, images preserved).
✅ Keeps Docker Desktop 4.79.0 (no downgrade needed, no uninstall and reinstall from scratch !).
✅ maybe more than 100x faster than Docker Support’s recommended reinstall.
---
🔍 Why This Works

Docker Desktop 4.79.0 has a bug with GPU runtimes in WSL2.
The nvidia-container-runtime in daemon.json causes a startup loop if the NVIDIA Toolkit is misconfigured or missing.
Removing the runtime references forces Docker to use the default runc runtime, avoiding the conflict.
---
🛡️ Prevention

Avoid mixing NVIDIA Container Toolkit with Docker Desktop 4.79.0 in WSL2.
If you need GPU support, install the toolkit after ensuring Docker Desktop is stable.
Disable auto-updates in Docker Desktop (Settings → General).
...
💡 FAQ
❓ Will this delete my containers/volumes?
No! This only removes the NVIDIA runtime references. Your data (containers, volumes, images) is safe.
❓ Can I reinstall NVIDIA Container Toolkit later?
Yes! Follow the official guide after Docker Desktop is stable.
❓ Why does this happen with Docker Desktop 4.79.0?
Docker Desktop 4.79.0 introduced changes in GPU runtime handling that conflict with some NVIDIA Toolkit versions.
❓ How do I check if NVIDIA Container Toolkit is installed?
bash :
dpkg -l | grep nvidia
---
🛠️ Automated Fix Script (Optional theorical script, i did it manually, step by step, not scripted, but... Try to make your script in case you have to face such issue ! OR, do not upgrade quickly to next last versions combination !!!)

If you want to automate the fix, run this script in WSL (Ubuntu-22.04):
bash
#!/bin/bash
# fix-docker-nvidia.sh

echo "🔧 Removing NVIDIA Container Toolkit..."
sudo apt remove --purge -y nvidia-container-toolkit libnvidia-container-tools libnvidia-container1 nvidia-container-toolkit-base
sudo apt autoremove -y

echo "📝 Cleaning daemon.json in WSL..."
echo "{}" | sudo tee /etc/docker/daemon.json

echo "✅ Done! Now clean Docker Desktop's daemon.json in Windows and restart Docker Desktop."
---
Usage:
bash
chmod +x fix-docker-nvidia.sh
./fix-docker-nvidia.sh
or make your own ! I decline any responsability in case of unfortunate problem as I do not know Your config !
📢 Contributing ,
You can APPLAUSE :) !
---
📜 License
This project is licensed under the MIT License

MIT License

   Copyright (c) 2026 Amine Mzerma

   Permission is hereby granted, free of charge, to any person obtaining a copy
   of this software and associated documentation files (the "Software"), to deal
   in the Software without restriction, including without limitation the rights
   to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
   copies of the Software, and to permit persons to whom the Software is
   furnished to do so, subject to the following conditions:

   The above copyright notice and this permission notice shall be included in all
   copies or substantial portions of the Software.

   THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
   IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
   FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
   AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
   LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
   OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
   SOFTWARE. The stroubleshooting solution did work on my Core ultra 7 265HX serie 2,
   with RTX 5060 ti 16 Gb, with intel performances package just installed one day
   before and updates made for the full docker components !



