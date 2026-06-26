# 🚀 Docker Desktop 4.79.0 Fix — NVIDIA Container Toolkit Conflict in WSL2

> **Fix for Docker Desktop 4.79.0 stuck on _"Starting the Docker Engine..."_ due to NVIDIA Container Toolkit conflicts in WSL2.**  
> **No data loss. No downgrade. No full reinstall.**

<p align="left">
  <img alt="Docker Desktop" src="https://img.shields.io/badge/Docker%20Desktop-4.79.0-2496ED?style=for-the-badge&logo=docker&logoColor=white">
  <img alt="Windows 11" src="https://img.shields.io/badge/Windows-11-0078D4?style=for-the-badge&logo=windows&logoColor=white">
  <img alt="WSL2" src="https://img.shields.io/badge/WSL2-Ubuntu%2022.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white">
  <img alt="NVIDIA" src="https://img.shields.io/badge/NVIDIA-GPU%20Runtime-76B900?style=for-the-badge&logo=nvidia&logoColor=white">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-black?style=for-the-badge">
</p>

---

## 📌 Problem

Docker Desktop **4.79.0** on **Windows 11 + WSL2** can get stuck on:

> **"Starting the Docker Engine..."**

In this case, the root issue was a conflict involving **NVIDIA Container Toolkit** and a stale `nvidia-container-runtime` reference still present in Docker configuration inside a WSL2-based setup.[page:1]

### Typical symptoms

- Docker Desktop hangs indefinitely during startup.[page:1]
- No obvious error appears in the UI.[page:1]
- Docker Engine settings may fail to apply cleanly.[page:1]
- WSL2 distros such as `docker-desktop` or `docker-desktop-data` may appear stopped or inconsistent.[page:1]
- Existing containers, volumes, and images are still there, but the engine does not come back correctly.[page:1]

---

## ✅ Solution

> **Tested and working**  
> **No data loss · No downgrade · No uninstall/reinstall from scratch**

The recovery was done in two layers:

1. **Clean the broken NVIDIA runtime state inside WSL**
2. **Clean the stale NVIDIA runtime declaration in Docker Desktop configuration on Windows**

---

## 1️⃣ Remove NVIDIA Container Toolkit from WSL

Run this in your Ubuntu WSL distro:

```bash
sudo apt remove --purge \
  nvidia-container-toolkit \
  nvidia-container-toolkit-base \
  libnvidia-container-tools \
  libnvidia-container1

sudo apt autoremove --purge
```

Then verify:

```bash
dpkg -l | grep -i nvidia || echo "No NVIDIA packages found"
```

If nothing is returned, the WSL side is clean.

---

## 2️⃣ Clean `daemon.json` in WSL

Check the current file:

```bash
sudo cat /etc/docker/daemon.json
```

Replace it with a minimal valid JSON config:

```bash
echo '{}' | sudo tee /etc/docker/daemon.json
```

This keeps a valid Docker daemon configuration file while removing any stale runtime declaration.

---

## 3️⃣ Clean Docker Desktop `daemon.json` on Windows

### Option A — From Docker Desktop UI
Open:

- **Docker Desktop**
- **Settings**
- **Docker Engine**

Remove all NVIDIA runtime references and keep a clean config.

### Option B — Better when Docker Desktop is looping
If Docker Desktop is stuck and keeps failing to save settings:

1. Quit Docker Desktop from the system tray.
2. Open **Task Manager**.
3. Kill all remaining Docker-related processes, including:
   - `Docker Desktop`
   - `Docker Desktop Backend`
   - `Docker Desktop Launcher`
   - `docker-agent.exe`
   - `docker-sandbox.exe`

Then edit the Windows Docker daemon config directly.

Typical path:

```text
C:\Users\<your_user>\.docker\daemon.json
```

Replace the content with:

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "default-runtime": "runc",
  "experimental": false,
  "insecure-registries": null,
  "log-driver": "json-file",
  "log-opts": {
    "max-file": "7",
    "max-size": "100m"
  },
  "registry-mirrors": null
}
```

### Important

Make a backup before editing, for example:

```text
daemon.json.bak
```

---

## 4️⃣ Restart WSL and Docker Desktop

From **Windows PowerShell or CMD**:

```powershell
wsl --shutdown
```

Then:

- relaunch your Ubuntu WSL distro,
- start Docker Desktop again from the Start menu,
- wait for the engine to come back.

---

## 🎯 Results

After applying this fix:

- ✅ Docker Desktop starts again
- ✅ Existing containers come back
- ✅ Volumes and images remain intact
- ✅ No downgrade required
- ✅ No reinstall required
- ✅ Recovery is dramatically faster than a full reset

In the tested case, GPU-related application stacks and standard web stacks both came back after the cleanup.[page:1]

---

## 🔍 Why this works

The issue was not the containers themselves.  
The issue was a **runtime configuration drift**:

- WSL had already lost or removed the NVIDIA toolkit state,
- but Docker Desktop still referenced `nvidia-container-runtime`,
- so the engine kept looping instead of falling back cleanly to `runc`.[page:1]

Removing the stale NVIDIA runtime reference forces Docker back onto its default runtime and clears the startup conflict.[page:1]

---

## 🛡️ Prevention

To reduce the chance of hitting this again:

- Avoid mixing partial NVIDIA runtime changes with a live Docker Desktop update.
- If Docker Desktop becomes unstable, do not keep applying UI settings blindly while it loops.
- Prefer a clean cold edit of `daemon.json` when the backend is stuck.
- Reintroduce GPU/runtime changes only after Docker Desktop is stable again.

---

## 💡 FAQ

### Will this delete my containers, volumes, or images?

**No.**  
This fix removes stale NVIDIA runtime references and repairs daemon configuration. It does **not** wipe Docker data.[page:1]

### Can NVIDIA Container Toolkit be reinstalled later?

**Yes.**  
But it should be done carefully and only after Docker Desktop is stable again.

### Why not just reinstall Docker Desktop?

Because in this case the platform was recoverable **without destructive reset**, and the existing workloads were preserved.[page:1]

### How do I check if NVIDIA packages are still present in WSL?

```bash
dpkg -l | grep -i nvidia
```

---

## 🛠️ Optional automation

This fix was performed **manually**, step by step.  
That was intentional.

A script could automate parts of it, but every environment is different, especially with GPU, WSL2, and Docker Desktop mixed together. If you script it, do it carefully and at your own risk.

Example starter script:

```bash
#!/bin/bash

echo "Removing NVIDIA Container Toolkit..."
sudo apt remove --purge -y \
  nvidia-container-toolkit \
  nvidia-container-toolkit-base \
  libnvidia-container-tools \
  libnvidia-container1

sudo apt autoremove -y

echo "Cleaning daemon.json in WSL..."
echo '{}' | sudo tee /etc/docker/daemon.json

echo "Done. Now clean Docker Desktop daemon.json on Windows and restart Docker Desktop."
```

---

## 🤝 Contributing

If this helped you recover Docker Desktop without reinstalling everything, a star is always appreciated.

---

## 📜 License

This project is licensed under the **MIT License**.

---

## 🧪 Tested context

This troubleshooting path worked on a live setup involving:

- Windows 11
- Docker Desktop 4.79.0
- WSL2 / Ubuntu 22.04
- NVIDIA GPU stack
- Existing running workloads to preserve

Your environment may differ, so apply carefully and always back up config files before editing.
