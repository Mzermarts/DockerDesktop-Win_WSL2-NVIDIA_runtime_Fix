# 🚀 Docker Desktop 4.79.0 — NVIDIA Container Toolkit Conflict Fix (WSL2)

> **Fix for Docker Desktop 4.79.0 stuck on *"Starting the Docker Engine…"* due to NVIDIA Container Toolkit conflicts in WSL2.**
> **No data loss · No downgrade · No full reinstall.**

---

[![Docker Desktop](https://img.shields.io/badge/Docker%20Desktop-4.79.0-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/products/docker-desktop/)
[![Windows 11](https://img.shields.io/badge/Windows-11-0078D4?style=for-the-badge&logo=windows&logoColor=white)](https://www.microsoft.com/windows/windows-11)
[![WSL2](https://img.shields.io/badge/WSL2-Ubuntu%2022.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com/wsl)
[![NVIDIA](https://img.shields.io/badge/NVIDIA-GPU%20Runtime-76B900?style=for-the-badge&logo=nvidia&logoColor=white)](https://docs.nvidia.com/datacenter/cloud-native/)
[![Blackwell](https://img.shields.io/badge/GPU-RTX%205060%20Ti%20Blackwell-76B900?style=for-the-badge&logo=nvidia&logoColor=white)](https://www.nvidia.com/en-us/geforce/graphics-cards/50-series/)
[![NCT](https://img.shields.io/badge/NCT%201.19.x-Regression-red?style=for-the-badge)](https://github.com/NVIDIA/nvidia-container-toolkit/releases)
[![License](https://img.shields.io/badge/License-MIT-black?style=for-the-badge)](LICENSE)

---

## 🌐 Language / Langue

- 🇬🇧 [English version](#-english-version)
- 🇫🇷 [Version française](#-version-française)

---

---

# 🇬🇧 English version

> ⚠️ **Before proceeding, please read the [Disclaimer & Limitation of Liability].**
> This fix is provided "as is", without warranty of any kind. Use at your own risk.

---

## 📋 Table of Contents

- [Context & Hardware](#-context--hardware)
- [Root Cause](#-root-cause)
- [Symptoms](#-symptoms)
- [Fix — Step by Step](#-fix--step-by-step)
  - [Step 1 — Remove NVIDIA Container Toolkit from WSL](#1️⃣-step-1--remove-nvidia-container-toolkit-from-wsl)
  - [Step 2 — Clean daemon.json in WSL](#2️⃣-step-2--clean-daemonjson-in-wsl)
  - [Step 3 — Clean daemon.json in Docker Desktop (Windows)](#3️⃣-step-3--clean-daemonjson-in-docker-desktop-windows)
  - [Step 4 — Restart WSL and Docker Desktop](#4️⃣-step-4--restart-wsl-and-docker-desktop)
- [Results](#-results)
- [Why this works](#-why-this-works)
- [Prevention](#️-prevention)
- [Optional automation script](#️-optional-automation-script)
- [FAQ](#-faq)
- [Contributing & References](#-contributing--references)

---

## 🖥️ Context & Hardware

This fix was validated on the following configuration:

| Component | Version / Model |
|---|---|
| OS | Windows 11 (latest) |
| Docker Desktop | 4.79.0 |
| WSL2 distro | Ubuntu 22.04 |
| GPU | **NVIDIA RTX 5060 Ti (Blackwell architecture)** |
| NVIDIA Container Toolkit | **1.19.x** ← regression version |
| Workloads preserved | GPU AI stacks + standard web stacks |

> ⚠️ **Important note on NVIDIA Container Toolkit 1.19.x**
>
> Version 1.19.x introduced a **regression specifically affecting Blackwell-architecture GPUs** (RTX 50-series).
> This version produces configuration drift that breaks Docker Desktop startup in WSL2 environments.
> Downgrading to **1.17.9-1** resolves GPU container access — but this README focuses on a **different
> scenario**: recovering Docker Desktop when it is stuck on startup due to stale NVIDIA runtime references,
> **without any downgrade at all**.

---

## 🔍 Root Cause

The issue is not the containers themselves.
The issue is **runtime configuration drift** between WSL2 and Docker Desktop:

```
WSL2 state              Docker Desktop config
─────────────────       ─────────────────────────────────
NVIDIA toolkit          daemon.json still references
removed / broken   ──►  "nvidia-container-runtime"
                        → Engine loops, never starts
```

Docker Desktop kept referencing `nvidia-container-runtime` in its daemon configuration,
while WSL had already lost or removed the NVIDIA toolkit state.
Instead of falling back cleanly to `runc`, the engine loops indefinitely.

---

## 🚨 Symptoms

- Docker Desktop hangs on **"Starting the Docker Engine…"** indefinitely
- No visible error in the UI
- Docker Engine settings fail to apply or save
- WSL2 distros (`docker-desktop`, `docker-desktop-data`) appear stopped or inconsistent
- Existing containers, volumes, and images are still present but the engine will not come up

---

## 🛠️ Fix — Step by Step

> 💾 **Before you start**: back up your Docker daemon config files.
>
> ```
> # Windows (PowerShell)
> Copy-Item "$env:USERPROFILE\.docker\daemon.json" "$env:USERPROFILE\.docker\daemon.json.bak"
> ```

---

### 1️⃣ Step 1 — Remove NVIDIA Container Toolkit from WSL

Open your Ubuntu WSL2 terminal and run:

```bash
sudo apt remove --purge \
  nvidia-container-toolkit \
  nvidia-container-toolkit-base \
  libnvidia-container-tools \
  libnvidia-container1

sudo apt autoremove --purge
```

Verify the WSL side is clean:

```bash
dpkg -l | grep -i nvidia || echo "✅ No NVIDIA packages found — WSL side is clean"
```

---

### 2️⃣ Step 2 — Clean `daemon.json` in WSL

Check the current state:

```bash
sudo cat /etc/docker/daemon.json
```

Replace with a minimal valid JSON config:

```bash
echo '{}' | sudo tee /etc/docker/daemon.json
```

This keeps a valid Docker daemon configuration file while removing all stale runtime declarations.

---

### 3️⃣ Step 3 — Clean `daemon.json` in Docker Desktop (Windows)

#### Option A — From the Docker Desktop UI

*(Use this only if Docker Desktop is not currently stuck in a loop)*

**Settings → Docker Engine** → remove all NVIDIA runtime references → Apply & Restart.

---

#### Option B — Direct file edit *(recommended when Docker Desktop is looping)*

**1. Fully quit Docker Desktop**

Right-click the system tray icon → **Quit Docker Desktop**.

**2. Kill all remaining Docker processes**

Open **Task Manager** and end all of the following:

| Process name |
|---|
| `Docker Desktop` |
| `Docker Desktop Backend` |
| `Docker Desktop Launcher` |
| `docker-agent.exe` |
| `docker-sandbox.exe` |

Or from PowerShell (run as Administrator):

```powershell
Get-Process | Where-Object { $_.Name -like "*docker*" } | Stop-Process -Force
```

**3. Edit the daemon config directly**

File location:

```
C:\Users\<your_user>\.docker\daemon.json
```

Replace the entire content with:

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

> The key fix is `"default-runtime": "runc"` — this explicitly tells Docker to stop looking
> for `nvidia-container-runtime` and fall back to the standard runtime.

---

### 4️⃣ Step 4 — Restart WSL and Docker Desktop

From **Windows PowerShell or CMD**:

```powershell
wsl --shutdown
```

Then:

1. Relaunch your Ubuntu WSL2 terminal
2. Start Docker Desktop from the Start menu
3. Wait for the engine to come up — it should no longer loop

---

## 🎯 Results

After applying this fix:

| Check | Status |
|---|---|
| Docker Desktop starts | ✅ |
| Existing containers restored | ✅ |
| Volumes intact | ✅ |
| Images intact | ✅ |
| No downgrade required | ✅ |
| No reinstall required | ✅ |
| Recovery time vs full reset | ✅ Dramatically faster |

> GPU-related AI stacks and standard web stacks both came back after the cleanup.

---

## 💡 Why this works

Docker Desktop uses `daemon.json` to know which container runtime to invoke.
When `nvidia-container-runtime` is listed there but is no longer available in WSL,
the engine startup fails silently in a loop.

Resetting `daemon.json` to a clean state with `"default-runtime": "runc"` removes
the missing dependency and lets Docker start normally.
NVIDIA GPU support can then be carefully reintroduced once Docker Desktop is stable.

---

## 🛡️ Prevention

- Avoid mixing partial NVIDIA runtime changes with a live Docker Desktop update
- If Docker Desktop becomes unstable, do **not** keep applying UI settings while it loops — it will not save correctly
- Prefer a **cold edit** of `daemon.json` (Docker Desktop fully stopped) when the backend is stuck
- Reintroduce GPU/runtime configuration only after Docker Desktop is confirmed stable
- On **Blackwell GPUs (RTX 50-series)**: avoid NVIDIA Container Toolkit **1.19.x** — use **1.17.9-1** instead

---

## ⚙️ Optional automation script

This fix was designed to be performed **manually and step by step** — intentionally.
Every environment is different, especially with GPU, WSL2, and Docker Desktop combined.

If you want to automate the WSL cleanup part:

```bash
#!/bin/bash
# docker-nvidia-wsl-cleanup.sh
# Run inside your Ubuntu WSL2 distro
# Always back up config files before running

set -e

echo "🧹 Removing NVIDIA Container Toolkit from WSL..."
sudo apt remove --purge -y \
  nvidia-container-toolkit \
  nvidia-container-toolkit-base \
  libnvidia-container-tools \
  libnvidia-container1 2>/dev/null || echo "Some packages were not installed — skipping"

sudo apt autoremove -y

echo "🧹 Cleaning WSL daemon.json..."
echo '{}' | sudo tee /etc/docker/daemon.json

echo ""
echo "✅ WSL side is clean."
echo "➡️  Now clean Docker Desktop daemon.json on Windows (see README Step 3)"
echo "➡️  Then run: wsl --shutdown"
echo "➡️  Then restart Docker Desktop"
```

> ⚠️ The Windows-side `daemon.json` edit must be done manually — a WSL script cannot safely
> write to Windows user profile paths.

---

## ❓ FAQ

**Will this delete my containers, volumes, or images?**
No. This fix removes stale runtime references and repairs daemon configuration only. Docker data is not touched.

**Can NVIDIA Container Toolkit be reinstalled later?**
Yes — but only after Docker Desktop is fully stable again, and carefully.
On Blackwell GPUs, use version **1.17.9-1**, not 1.19.x.

**Why not just reinstall Docker Desktop?**
Because the platform was fully recoverable without a destructive reset, and all existing workloads were preserved.
A reinstall would have been unnecessary data risk.

**How do I check if NVIDIA packages are still present in WSL?**

```bash
dpkg -l | grep -i nvidia
```

If nothing is returned, WSL is clean.

**What if Docker Desktop still loops after this fix?**
Double-check that all Docker processes were fully killed before editing `daemon.json` on Windows.
A partially running `docker-agent.exe` can overwrite the config on exit.

---

## 🤝 Contributing & References

If this fix saved your workloads, a ⭐ is always appreciated.

**Related issues where this scenario appears:**

- [microsoft/WSL #9962](https://github.com/microsoft/WSL/issues/9962) — GPU access blocked in WSL2 containers
- [docker/for-win #11246](https://github.com/docker/for-win/issues/11246) — WSL2 GPU support from Docker Desktop Windows
- [NVIDIA/nvidia-container-toolkit #665](https://github.com/NVIDIA/nvidia-container-toolkit/issues/665) — daemon.json not configured correctly on Windows + WSL2

---

## 📜 License

MIT License — see [LICENSE](LICENSE)

---

---

# 🇫🇷 Version française

> ⚠️ **Avant de procéder, merci de lire le [Disclaimer & Clause de non-responsabilité](DISCLAIMER.md).**
> Ce correctif est fourni « en l'état », sans garantie d'aucune sorte. Utilisation à vos propres risques.

---

## 📋 Table des matières

- [Contexte & matériel](#️-contexte--matériel)
- [Cause racine](#-cause-racine)
- [Symptômes](#-symptômes)
- [Correctif étape par étape](#️-correctif-étape-par-étape)
  - [Étape 1 — Supprimer NVIDIA Container Toolkit de WSL](#1️⃣-étape-1--supprimer-nvidia-container-toolkit-de-wsl)
  - [Étape 2 — Nettoyer daemon.json dans WSL](#2️⃣-étape-2--nettoyer-daemonjson-dans-wsl)
  - [Étape 3 — Nettoyer daemon.json côté Docker Desktop (Windows)](#3️⃣-étape-3--nettoyer-daemonjson-côté-docker-desktop-windows)
  - [Étape 4 — Redémarrer WSL et Docker Desktop](#4️⃣-étape-4--redémarrer-wsl-et-docker-desktop)
- [Résultats](#-résultats)
- [Pourquoi ça fonctionne](#-pourquoi-ça-fonctionne)
- [Prévention](#️-prévention)
- [Script d'automatisation optionnel](#️-script-dautomatisation-optionnel)
- [FAQ](#-faq-1)

---

## 🖥️ Contexte & matériel

Ce correctif a été validé sur la configuration suivante :

| Composant | Version / Modèle |
|---|---|
| OS | Windows 11 (dernière version) |
| Docker Desktop | 4.79.0 |
| Distro WSL2 | Ubuntu 22.04 |
| GPU | **NVIDIA RTX 5060 Ti (architecture Blackwell)** |
| NVIDIA Container Toolkit | **1.19.x** ← version en régression |
| Charges de travail préservées | Stacks IA GPU + stacks web standard |

> ⚠️ **Note importante sur NVIDIA Container Toolkit 1.19.x**
>
> La version 1.19.x introduit une **régression spécifique aux GPU d'architecture Blackwell** (série RTX 50).
> Cette version produit une dérive de configuration qui casse le démarrage de Docker Desktop dans les
> environnements WSL2. La rétrogradation vers **1.17.9-1** résout l'accès GPU dans les conteneurs —
> mais ce README traite d'un **scénario différent** : récupérer Docker Desktop bloqué au démarrage
> à cause de références NVIDIA obsolètes, **sans aucune rétrogradation**.

---

## 🔍 Cause racine

Le problème ne vient pas des conteneurs eux-mêmes.
Le problème est une **dérive de configuration du runtime** entre WSL2 et Docker Desktop :

```
État WSL2                   Config Docker Desktop
─────────────────           ─────────────────────────────────
NVIDIA toolkit              daemon.json référence toujours
supprimé / cassé    ──►     "nvidia-container-runtime"
                            → Le moteur boucle, ne démarre plus
```

Docker Desktop continuait à référencer `nvidia-container-runtime` dans sa configuration daemon,
alors que WSL avait déjà perdu ou supprimé l'état du NVIDIA toolkit.
Au lieu de basculer proprement sur `runc`, le moteur boucle indéfiniment.

---

## 🚨 Symptômes

- Docker Desktop reste bloqué sur **« Starting the Docker Engine… »** indéfiniment
- Aucune erreur visible dans l'interface
- Les paramètres du Docker Engine échouent à s'appliquer ou à se sauvegarder
- Les distros WSL2 (`docker-desktop`, `docker-desktop-data`) apparaissent arrêtées ou incohérentes
- Les conteneurs, volumes et images existants sont toujours là, mais le moteur ne remonte pas

---

## 🛠️ Correctif étape par étape

> 💾 **Avant de commencer** : sauvegardez vos fichiers de configuration Docker.
>
> ```powershell
> # Windows (PowerShell)
> Copy-Item "$env:USERPROFILE\.docker\daemon.json" "$env:USERPROFILE\.docker\daemon.json.bak"
> ```

---

### 1️⃣ Étape 1 — Supprimer NVIDIA Container Toolkit de WSL

Dans votre terminal Ubuntu WSL2 :

```bash
sudo apt remove --purge \
  nvidia-container-toolkit \
  nvidia-container-toolkit-base \
  libnvidia-container-tools \
  libnvidia-container1

sudo apt autoremove --purge
```

Vérification :

```bash
dpkg -l | grep -i nvidia || echo "✅ Aucun paquet NVIDIA trouvé — WSL est propre"
```

---

### 2️⃣ Étape 2 — Nettoyer `daemon.json` dans WSL

Vérification de l'état actuel :

```bash
sudo cat /etc/docker/daemon.json
```

Remplacement par un JSON minimal valide :

```bash
echo '{}' | sudo tee /etc/docker/daemon.json
```

---

### 3️⃣ Étape 3 — Nettoyer `daemon.json` côté Docker Desktop (Windows)

#### Option A — Via l'interface Docker Desktop

*(Uniquement si Docker Desktop n'est pas en boucle)*

**Settings → Docker Engine** → supprimer toutes les références NVIDIA → Apply & Restart.

---

#### Option B — Édition directe du fichier *(recommandée quand Docker Desktop boucle)*

**1. Quitter Docker Desktop complètement**

Clic droit sur l'icône dans la barre système → **Quit Docker Desktop**.

**2. Tuer tous les processus Docker restants**

Dans le **Gestionnaire des tâches**, terminer tous les processus suivants :

| Nom du processus |
|---|
| `Docker Desktop` |
| `Docker Desktop Backend` |
| `Docker Desktop Launcher` |
| `docker-agent.exe` |
| `docker-sandbox.exe` |

Ou depuis PowerShell (en administrateur) :

```powershell
Get-Process | Where-Object { $_.Name -like "*docker*" } | Stop-Process -Force
```

**3. Éditer le fichier de configuration directement**

Emplacement du fichier :

```
C:\Users\<votre_utilisateur>\.docker\daemon.json
```

Remplacer l'intégralité du contenu par :

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

> La correction clé est `"default-runtime": "runc"` — cela indique explicitement à Docker
> d'arrêter de chercher `nvidia-container-runtime` et de revenir au runtime standard.

---

### 4️⃣ Étape 4 — Redémarrer WSL et Docker Desktop

Depuis **PowerShell ou CMD Windows** :

```powershell
wsl --shutdown
```

Ensuite :

1. Relancer votre terminal Ubuntu WSL2
2. Démarrer Docker Desktop depuis le menu Démarrer
3. Attendre que le moteur remonte — il ne devrait plus boucler

---

## 🎯 Résultats

Après application du correctif :

| Vérification | Statut |
|---|---|
| Docker Desktop démarre | ✅ |
| Conteneurs existants restaurés | ✅ |
| Volumes intacts | ✅ |
| Images intactes | ✅ |
| Aucune rétrogradation nécessaire | ✅ |
| Aucune réinstallation nécessaire | ✅ |
| Temps de récupération vs reset complet | ✅ Drastiquement plus rapide |

---

## 💡 Pourquoi ça fonctionne

Docker Desktop utilise `daemon.json` pour savoir quel runtime de conteneur invoquer.
Quand `nvidia-container-runtime` y est déclaré mais n'est plus disponible dans WSL,
le démarrage du moteur échoue silencieusement en boucle.

Réinitialiser `daemon.json` à un état propre avec `"default-runtime": "runc"` supprime
la dépendance manquante et permet à Docker de démarrer normalement.
Le support GPU NVIDIA peut ensuite être réintroduit prudemment une fois Docker Desktop stable.

---

## 🛡️ Prévention

- Éviter de mélanger des modifications partielles du runtime NVIDIA avec une mise à jour live de Docker Desktop
- Si Docker Desktop devient instable, ne **pas** continuer à appliquer des paramètres via l'UI en boucle — ils ne se sauvegarderont pas correctement
- Préférer une **édition à froid** de `daemon.json` (Docker Desktop complètement arrêté) quand le backend est bloqué
- Réintroduire la configuration GPU/runtime uniquement après confirmation que Docker Desktop est stable
- Sur **GPU Blackwell (RTX série 50)** : éviter NVIDIA Container Toolkit **1.19.x** — utiliser **1.17.9-1** à la place

---

## ⚙️ Script d'automatisation optionnel

Ce correctif a été conçu pour être appliqué **manuellement, étape par étape** — intentionnellement.
Chaque environnement est différent, surtout avec GPU, WSL2 et Docker Desktop combinés.

Pour automatiser la partie nettoyage WSL :

```bash
#!/bin/bash
# docker-nvidia-wsl-cleanup.sh
# À exécuter dans votre distro Ubuntu WSL2
# Sauvegardez toujours vos fichiers de config avant d'exécuter

set -e

echo "🧹 Suppression du NVIDIA Container Toolkit de WSL..."
sudo apt remove --purge -y \
  nvidia-container-toolkit \
  nvidia-container-toolkit-base \
  libnvidia-container-tools \
  libnvidia-container1 2>/dev/null || echo "Certains paquets n'étaient pas installés — ignoré"

sudo apt autoremove -y

echo "🧹 Nettoyage de daemon.json dans WSL..."
echo '{}' | sudo tee /etc/docker/daemon.json

echo ""
echo "✅ Côté WSL propre."
echo "➡️  Nettoyez maintenant daemon.json dans Docker Desktop sur Windows (voir Étape 3)"
echo "➡️  Puis exécutez : wsl --shutdown"
echo "➡️  Puis redémarrez Docker Desktop"
```

> ⚠️ L'édition de `daemon.json` côté Windows doit être faite manuellement — un script WSL
> ne peut pas écrire de manière fiable dans les chemins du profil utilisateur Windows.

---

## ❓ FAQ

**Est-ce que ça va supprimer mes conteneurs, volumes ou images ?**
Non. Ce correctif supprime uniquement les références de runtime obsolètes et répare la configuration daemon. Les données Docker ne sont pas touchées.

**Peut-on réinstaller NVIDIA Container Toolkit ensuite ?**
Oui — mais seulement après que Docker Desktop est pleinement stable, et avec précaution.
Sur GPU Blackwell, utiliser la version **1.17.9-1**, pas la 1.19.x.

**Pourquoi ne pas simplement réinstaller Docker Desktop ?**
Parce que la plateforme était entièrement récupérable sans reset destructif, et toutes les charges de travail existantes ont été préservées. Une réinstallation aurait constitué un risque de perte de données inutile.

**Comment vérifier si des paquets NVIDIA sont encore présents dans WSL ?**

```bash
dpkg -l | grep -i nvidia
```

Si rien ne s'affiche, WSL est propre.

**Et si Docker Desktop boucle encore après ce correctif ?**
Vérifiez que tous les processus Docker ont bien été tués avant d'éditer `daemon.json` sur Windows.
Un `docker-agent.exe` partiellement actif peut réécrire la config à sa fermeture.

---

## 📜 Licence

MIT License — voir [LICENSE](LICENSE)

---

*README rédigé et structuré avec l'aide de [Claude](https://claude.ai) (Anthropic) — juin 2026.*
