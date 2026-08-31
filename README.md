
🇬🇧 English | [🇮🇹 Italiano](README.it.md)

# Lab Notes — Local AI with llama.cpp and Docker

> **This document is my personal technical diary.**
> I write it for myself, so I can retrace every step of the project. It includes inline explanations for every term or procedure that might not be immediately obvious. If you're reading this on GitHub, welcome: this is an account of how I installed and configured a local LLM server with GPU acceleration on Kali Linux, including every error I ran into and how I fixed it.

---

## Table of Contents

1. [Project Goal](#1-project-goal)
2. [Hardware and Software Used](#2-hardware-and-software-used)
3. [Project Structure](#3-project-structure)
4. [Phase 1 — Installing Docker](#4-phase-1--installing-docker)
5. [Phase 2 — Configuring Docker Permissions](#5-phase-2--configuring-docker-permissions)
6. [Phase 3 — Installing the NVIDIA Container Toolkit](#6-phase-3--installing-the-nvidia-container-toolkit)
7. [Phase 4 — Updating the NVIDIA Driver](#7-phase-4--updating-the-nvidia-driver)
8. [Phase 5 — Configuring the Project Files](#8-phase-5--configuring-the-project-files)
9. [Phase 6 — Starting the Server](#9-phase-6--starting-the-server)
10. [Errors Encountered and Solutions](#10-errors-encountered-and-solutions)
11. [Everyday Commands](#11-everyday-commands)
12. [Final Notes and Lessons Learned](#12-final-notes-and-lessons-learned)

---

## 1. Project Goal

The goal is to run an LLM (Large Language Model — an AI model for language, like the ones behind ChatGPT) locally on my computer, using the NVIDIA GPU to accelerate inference (the process by which the model generates responses).

The chosen software is **llama.cpp**, an open-source inference engine that supports models in GGUF format (a compressed format optimized for local execution). llama.cpp runs inside a **Docker container** (an isolated, reproducible environment, similar to a lightweight virtual machine) to simplify management.

The end result is an HTTP server accessible at `http://localhost:8081` that exposes an API usable by any client (browser, app, Python script).

---

## 2. Hardware and Software Used

| Component | Detail |
|---|---|
| CPU | Intel i5-7500 |
| RAM | 16 GB |
| GPU | NVIDIA GeForce RTX 3060 (12 GB VRAM) |
| Storage | NVMe 465 GB |
| Operating system | Kali Linux |
| NVIDIA driver | 595.80 |
| CUDA | 13.2 |
| Docker | 28.5.2 |
| llama.cpp | image `ghcr.io/ggml-org/llama.cpp:full-cuda` |
| Models used | Gemma-4 E4B (Q4_K_XL), Dolphin Llama3.1 8B (Q8_0) |

> **CUDA** is a platform developed by NVIDIA that lets programs use the GPU for general-purpose computation (not just graphics). The CUDA version depends on the installed driver: newer drivers support higher CUDA versions.

---

## 3. Project Structure

```
~/Progetti/AI-local/
├── docker-compose.yml   # Defines how to start the container
├── models.ini           # LLM model configuration
└── gguf/
    ├── gemma-4-E4B-it-UD-Q4_K_XL.gguf
    └── Dolphin3.0-Llama3.1-8B-Q8_0.gguf
```

Create the structure from scratch:

```bash
mkdir -p ~/Progetti/AI-local/gguf
cd ~/Progetti/AI-local
```

> `mkdir -p` creates the folder and all necessary intermediate folders. `-p` stands for "parents".

---

## 4. Phase 1 — Installing Docker

### Initial problem

Docker was not installed. The system only had `docker-cli` available in the repos, but this package contains only the client (the tool that sends commands) without the daemon (the process that runs containers). Without the daemon, Docker doesn't work.

### Solution

Install `docker.io`, which includes everything needed:

```bash
sudo apt update
sudo apt install docker.io
```

> `apt` is the Debian/Kali package manager. `sudo` runs the command as administrator. `docker.io` is the full package that includes the `dockerd` daemon, the client, and `containerd` (the underlying runtime).

Start and check the daemon:

```bash
sudo systemctl start docker
sudo systemctl status docker
```

> `systemctl` is the tool for managing system services on Linux. `start` starts the service, `status` shows its state. The Docker daemon is called `docker.service`.

Expected output from `status`:
```
Active: active (running)
```

Enable Docker to start automatically at boot (optional but recommended):

```bash
sudo systemctl enable docker
```

---

## 5. Phase 2 — Configuring Docker Permissions

### Problem

Running `docker-compose up -d` without privileges produced this error:

```
permission denied while trying to connect to the Docker daemon socket
at unix:///var/run/docker.sock
```

### Explanation

The Docker daemon communicates through a special file called a **socket** (`/var/run/docker.sock`). By default, only the `root` user and users in the `docker` group can access it. The `kali` user was not in that group.

### Solution

```bash
sudo usermod -aG docker $USER
newgrp docker
```

> `usermod` modifies a user's settings. `-a` means "append" (without removing existing groups). `-G docker` specifies the group to add. `$USER` is an environment variable that automatically holds the current username.
>
> `newgrp docker` opens a new shell with the `docker` group already active, without needing to log out. It's equivalent to logging back in, but faster. **Note:** it must be run every time a new terminal is opened, until a full logout and login is done.

Verify the group is active:

```bash
groups
```

`docker` should appear in the output.

---

## 6. Phase 3 — Installing the NVIDIA Container Toolkit

### Problem

Even with Docker working, starting the container with GPU support produced:

```
could not select device driver "nvidia" with capabilities: [[gpu]]
```

### Explanation

Docker alone doesn't know how to talk to the NVIDIA GPU. The **NVIDIA Container Toolkit** is needed — a software layer that bridges Docker and the NVIDIA driver installed on the system.

### Solution

Add the official NVIDIA repository and install the toolkit:

```bash
# Download and save the NVIDIA repository's GPG key
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# Add the repository to apt's source list
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Install
sudo apt update && sudo apt install -y nvidia-container-toolkit
```

> **GPG key**: a cryptographic signature that guarantees the downloaded packages really come from NVIDIA and haven't been tampered with.
>
> `curl` is a tool for downloading content from URLs. `-fsSL` means: fail silently on error (`f`), don't show a progress bar (`s`), follow redirects (`L`).
>
> `gpg --dearmor` converts the key from text format to the binary format used by apt.
>
> `tee` writes the output to both a file and the screen at the same time.
>
> `sed` is a text-editing tool. Here it inserts the reference to the GPG key inside the repository file.

Register the NVIDIA runtime with Docker and restart:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

> `nvidia-ctk` is the toolkit's configuration tool. `runtime configure` modifies `/etc/docker/daemon.json`, adding the `nvidia` runtime. Without this step, Docker doesn't know the runtime exists.

---

## 7. Phase 4 — Updating the NVIDIA Driver

### Problem

Even after installing the toolkit, the container failed to start with:

```
nvidia-container-cli: requirement error: unsatisfied condition: cuda>=12.8,
please update your driver to a newer version
```

### Explanation

The `ghcr.io/ggml-org/llama.cpp:full-cuda` Docker image is built against CUDA 12.8+. The installed driver (version 550, CUDA 12.4) was too old. Kali Linux's repositories only had driver 550, so it couldn't be updated via `apt`.

### Checking the situation

```bash
nvidia-smi                          # shows current driver and CUDA version
apt-cache policy nvidia-driver      # shows the version available in the repos
apt-cache search nvidia-driver | grep -v lib  # lists available packages
```

### Solution: manual driver installation

Download the latest driver from the official NVIDIA site:
- URL: https://www.nvidia.com/en-us/drivers/
- Select: GeForce → GeForce RTX 30 Series → RTX 3060 → Linux 64-bit
- Version downloaded: **595.80** (released May 27, 2026)

Stop the graphical server (X11) before installation, because the driver can't be installed while the GPU is in use by the display:

```bash
sudo systemctl stop display-manager
```

> `display-manager` is the service that manages the graphical interface (login screen and desktop). Stopping it puts the system into text console mode (black screen with a blinking cursor — this is normal).

Run the installer:

```bash
sudo sh ~/Scaricati/NVIDIA-Linux-x86_64-595.80.run
```

During installation, the interactive wizard presented these choices:

| Question | Answer | Reason |
|---|---|---|
| Kernel module type | **MIT/GPL** | Open-source module, more compatible with modern Linux kernels. Recommended for RTX 30 series and later. |
| Disable Nouveau | **Yes** | Nouveau is the generic open-source driver for NVIDIA GPUs. It conflicts with the proprietary driver and must be disabled. |
| Rebuild initramfs | **Rebuild initramfs** | The initramfs is the image loaded by the kernel at boot. It must be rebuilt to include the Nouveau disablement, otherwise Nouveau reloads on reboot. |
| Install 32-bit libraries | **Yes** | Useful for legacy applications (e.g. Steam). Doesn't take up much space. |
| Update X11 configuration | **Yes** | Updates the display server's configuration file to use the NVIDIA driver on next boot. |

Reboot the system:

```bash
sudo reboot
```

Verify the installation:

```bash
nvidia-smi
```

Expected output: driver 595.80, CUDA Version 13.2.

> **Warning:** the driver installed this way is outside `apt`'s management. After Linux kernel updates it may need to be reinstalled with the same `.run` file.

---

## 8. Phase 5 — Configuring the Project Files

### docker-compose.yml

The `docker-compose.yml` file defines how Docker should start the container. Create it in `~/Progetti/AI-local/`:

```yaml
services:
  llama-server:
    image: ghcr.io/ggml-org/llama.cpp:full-cuda
    container_name: llama-server
    ports:
      - "8081:8081"
    volumes:
      - /home/kali/Progetti/AI-local/gguf/:/models
      - ./models.ini:/config.ini
    command: |
      --server
      --models-preset /config.ini
      --sleep-idle-seconds 600
      --parallel 1
      --port 8081
      --host 127.0.0.1
      --jinja
      --models-autoload
      --models-max 1
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: on-failure
```

**Explanation of the main fields:**

| Field | Meaning |
|---|---|
| `image` | The Docker image to use. `full-cuda` includes llama.cpp compiled with CUDA support. |
| `ports: "8081:8081"` | Maps container port 8081 to host port 8081. Format: `host:container`. |
| `volumes` | Mounts folders from the host computer into the container. The `gguf/` folder becomes `/models` inside the container. |
| `--models-preset /config.ini` | Uses the models.ini file to load model presets. |
| `--models-autoload` | Loads the model into memory only when the first request arrives (lazy loading). |
| `--models-max 1` | Keeps at most one model loaded in VRAM at a time. |
| `--sleep-idle-seconds 600` | Unloads the model from VRAM after 10 minutes of inactivity. |
| `capabilities: [gpu]` | Declares that the container needs GPU access. |
| `restart: on-failure` | Automatically restarts the container if it crashes. |

> **Network security note:** `--host 127.0.0.1` restricts the server so it only accepts connections from the local machine (the container/host itself), not from the network. With `--host 0.0.0.0` instead, the server would be reachable from any device on the same local network, and the API has no authentication — anyone on that network could send requests to the model.

### models.ini

The `models.ini` file configures the available models. Create it in `~/Progetti/AI-local/`:

```ini
[*] ; Global parameters, valid for all models
flash-attn = on
gpu-layers = 999
batch-size = 512
ubatch-size = 256
cache-type-k = q4_0
cache-type-v = q4_0
repeat-penalty = 1.0
cont-batching = true
slots = 1
fit = on

[Gemma-4 E4B] ; Preset specific to Gemma
model = /models/gemma-4-E4B-it-UD-Q4_K_XL.gguf
c = 16384
temp = 0.8

[Dolphin Llama3.1 8B] ; Preset specific to Dolphin
model = /models/Dolphin3.0-Llama3.1-8B-Q8_0.gguf
c = 16384
temp = 0.8
```

**Explanation of the main parameters:**

| Parameter | Meaning |
|---|---|
| `gpu-layers = 999` | Loads all model layers into VRAM. 999 means "as many as possible". |
| `flash-attn = on` | Enables Flash Attention, an optimized algorithm that reduces VRAM usage during inference. |
| `cache-type-k/v = q4_0` | Quantizes the KV cache (memory used for context) to 4-bit to save VRAM. |
| `c = 16384` | Context size: how many tokens (text units, roughly 0.75 words on average) the model can hold in memory during a conversation. 8192 is the minimum, 32768 is the reasonable maximum with 12GB VRAM. |
| `temp = 0.8` | Temperature: controls response "creativity". 0 = deterministic, 1 = highly varied. |
| `fit = on` | If the model doesn't fit entirely in VRAM, automatically loads the remaining layers into RAM. |

---

## 9. Phase 6 — Starting the Server

### First run (downloads the Docker image)

```bash
cd ~/Progetti/AI-local
newgrp docker
docker-compose up -d
```

The first run downloads the `full-cuda` image from GitHub Container Registry (~15 GB). Takes time depending on connection speed.

> `docker-compose up` starts the services defined in `docker-compose.yml`. `-d` stands for "detached": the container runs in the background and the terminal stays free.

### Checking the server is running

```bash
docker ps
```

Should show `llama-server` with status `Up`.

```bash
docker logs llama-server
```

Expected output:
```
srv  llama_server: router server is listening on http://127.0.0.1:8081
```

### Monitoring GPU usage

```bash
watch -n 1 nvidia-smi
```

> `watch -n 1` re-runs the command every second and refreshes the screen. Useful for watching VRAM usage in real time during inference. Exit with `Ctrl+C`.

With the Gemma-4 E4B model loaded, expected usage is around 3600-4000 MiB out of 12288 MiB total.

### Accessing the server

Open a browser and go to:
```
http://localhost:8081
```

The llama.cpp web interface lets you chat directly with the model.

---

## 10. Errors Encountered and Solutions

### Error 1 — `docker.service not found`

**Message:**
```
Failed to start docker.service: Unit docker.service not found.
```

**Cause:** only `docker-cli` was installed, not the full daemon.

**Solution:**
```bash
sudo apt install docker.io
sudo systemctl start docker
```

---

### Error 2 — `permission denied` on the Docker socket

**Message:**
```
permission denied while trying to connect to the Docker daemon socket
at unix:///var/run/docker.sock
```

**Cause:** the user was not in the `docker` group.

**Solution:**
```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

### Error 3 — `could not select device driver "nvidia"`

**Message:**
```
could not select device driver "nvidia" with capabilities: [[gpu]]
```

**Cause:** NVIDIA Container Toolkit not installed or not configured.

**Solution:**
```bash
# Install the toolkit (see Phase 3)
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
newgrp docker
```

---

### Error 4 — `cuda>=12.8` not satisfied

**Message:**
```
nvidia-container-cli: requirement error: unsatisfied condition: cuda>=12.8
```

**Cause:** NVIDIA driver version 550 (CUDA 12.4), too old for the current Docker image.

**Solution:** update the driver to version 595.80 via the official installer (see Phase 4).

---

### Error 5 — context size exceeded

**Message:**
```
request (10821 tokens) exceeds the available context size (8192 tokens)
```

**Cause:** the `c` parameter in `models.ini` was set to 8192, not enough for long conversations.

**Solution:** increase the context size in `models.ini`:
```ini
c = 16384
```
Then restart:
```bash
docker-compose restart
```

---

### Error 6 — wrong model filename

**Cause:** `models.ini` had `Q5_K_XL` written, but the actual file was `Q4_K_XL`.

**Solution:** check the exact filename:
```bash
ls ~/Progetti/AI-local/gguf/
```
Fix `models.ini` with the exact name.

---

## 11. Everyday Commands

### Starting the server

```bash
cd ~/Progetti/AI-local
newgrp docker
docker-compose up -d
```

### Stopping the server

```bash
cd ~/Progetti/AI-local
docker-compose down
```

### Restarting after changes to models.ini

```bash
docker-compose restart
```

### Watching logs in real time

```bash
docker logs -f llama-server
```

> `-f` stands for "follow": logs update in real time. Exit with `Ctrl+C`.

### Monitoring GPU

```bash
watch -n 1 nvidia-smi
```

### Checking active containers

```bash
docker ps
```

---

## 12. Final Notes and Lessons Learned

**Memory between sessions:** the model has no persistent memory. Every new chat starts from scratch. For ongoing projects, keep a `context.md` file with a project summary to paste at the start of every new session.

**Context size and VRAM:** with 12 GB of VRAM and the Gemma-4 E4B Q4 model, the reasonable maximum context size is around 32768 tokens. Higher values cause out-of-memory errors. Monitor with `nvidia-smi` during inference.

**NVIDIA driver outside apt:** driver 595.80 is installed manually. After Linux kernel updates, it may need to be reinstalled with:
```bash
sudo systemctl stop display-manager
sudo sh ~/Scaricati/NVIDIA-Linux-x86_64-595.80.run
sudo reboot
```

**newgrp docker:** must be run every time a new terminal is opened, until a full logout is done. After logging out and back in, the group membership is permanent and `newgrp` is no longer needed.

**Available models:**
- `Gemma-4 E4B Q4_K_XL` — compact model, good quality/speed ratio, optimized for Italian
- `Dolphin Llama3.1 8B Q8_0` — larger model, higher quality, uses more VRAM

With `--models-max 1`, only one model at a time is loaded into VRAM. Switching models in the UI automatically unloads the previous one.
