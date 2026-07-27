# Local AI with llama.cpp and Docker

Guide to install and run a local LLM (Large Language Model) server on Linux, with NVIDIA GPU acceleration via Docker.

---

## Requirements

### Hardware
- NVIDIA GPU with at least 6 GB VRAM (12 GB recommended for 8B+ models)
- System RAM: at least 16 GB

### Software
- Linux (tested on Kali Linux, works on any Debian/Ubuntu-based distro)
- NVIDIA drivers installed and working
- Docker
- NVIDIA Container Toolkit
- A model in `.gguf` format downloaded locally

---

## Project structure

```
AI-local/
├── docker-compose.yml
├── models.ini
└── gguf/
    ├── model-1.gguf
    └── model-2.gguf
```

- `docker-compose.yml` — configures the container: ports, volumes, and GPU access
- `models.ini` — defines available models and their parameters
- `gguf/` — folder where downloaded model files go

---

## 1. Install Docker

```bash
sudo apt update
sudo apt install docker.io docker-cli docker-compose
```

Start the daemon and enable it at boot:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

Add your user to the `docker` group to run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

> `newgrp docker` applies the group to the current session. To make it permanent without running it every time, log out and log back in.

---

## 2. Install the NVIDIA driver

The driver must support CUDA 12.8 or higher. Check the installed version:

```bash
nvidia-smi
```

If the driver is missing or the CUDA version is below 12.8, download it from [nvidia.com/drivers](https://www.nvidia.com/drivers) and install it:

```bash
sudo sh NVIDIA-Linux-x86_64-XXX.XX.run
```

During installation:
- Accept the license
- Enable Nouveau disabling (Nouveau is the default open-source driver, incompatible with CUDA)
- Rebuild initramfs when prompted

Verify after reboot:

```bash
nvidia-smi
```

---

## 3. Install NVIDIA Container Toolkit

This component allows Docker to pass the GPU through to containers.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit
```

Register the NVIDIA runtime with Docker:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

## 4. Download a model

Compatible models are in `.gguf` format. They can be found on [Hugging Face](https://huggingface.co).

Quantization selection guide based on available VRAM:

| Available VRAM | Recommended quantization |
|---|---|
| 6 GB | Q4_K_M |
| 8 GB | Q5_K_M |
| 12 GB | Q8_0 or Q4_K_XL |
| 24 GB+ | Q8_0 on 13B+ models |

Place the downloaded file in the `gguf/` folder.

---

## 5. Configure the files

### `docker-compose.yml`

```yaml
services:
  llama-server:
    image: ghcr.io/ggml-org/llama.cpp:full-cuda
    container_name: llama-server
    ports:
      - "8081:8081"
    volumes:
      - ./gguf/:/models
      - ./models.ini:/config.ini
    command: |
      --server
      --models-preset /config.ini
      --sleep-idle-seconds 600
      --parallel 1
      --port 8081
      --host 0.0.0.0
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

**Key parameters:**
- `ports` — port exposed on the host (format `host:container`)
- `volumes` — maps the local `gguf/` folder to `/models` inside the container
- `--sleep-idle-seconds 600` — unloads the model from VRAM after 10 minutes of inactivity
- `--models-max 1` — keeps only one model loaded in memory at a time

### `models.ini`

```ini
[*]
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

[Model Name]
model = /models/exact-file-name.gguf
c = 32768
temp = 0.8
```

**`[*]` section** — global parameters applied to all models:
- `gpu-layers = 999` — loads all layers into VRAM (use a lower number for partial CPU offloading)
- `cache-type-k/v = q4_0` — quantizes the KV cache, reduces VRAM usage
- `fit = on` — automatically adjusts layers to fit available VRAM

**Per-model section** — add one section per `.gguf` file in the `gguf/` folder:
- `model` — exact path to the file inside the container (always `/models/filename.gguf`)
- `c` — context size in tokens (32768 is a good default with 12 GB VRAM)
- `temp` — generation temperature (0.0 = deterministic, 1.0 = creative)

> The filename in `model =` must match the `.gguf` file name exactly, including capitalization.

---

## 6. Start the server

```bash
docker-compose up -d
```

Check that the container started correctly:

```bash
docker logs llama-server
```

The server is ready when the logs show a line like:

```
llama server listening at http://0.0.0.0:8081
```

Web interface available at: `http://localhost:8081`

---

## Useful commands

| Command | Function |
|---|---|
| `docker-compose up -d` | Start the server in the background |
| `docker-compose down` | Stop and remove the container |
| `docker-compose restart` | Restart the container (use after editing `models.ini`) |
| `docker logs llama-server` | Show container logs |
| `docker logs -f llama-server` | Follow logs in real time |
| `watch -n 1 nvidia-smi` | Monitor GPU usage every second |

---

## Update the Docker image

To use the latest version of llama.cpp:

```bash
docker-compose down
docker pull ghcr.io/ggml-org/llama.cpp:full-cuda
docker-compose up -d
```

---

## Add a new model

1. Download the `.gguf` file into the `gguf/` folder
2. Add a section to `models.ini`:
   ```ini
   [Model Name]
   model = /models/exact-file-name.gguf
   c = 32768
   temp = 0.8
   ```
3. Restart the container:
   ```bash
   docker-compose restart
   ```
