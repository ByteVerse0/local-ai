# Local LLM server with llama.cpp and Docker

A local, GPU-accelerated LLM server. It runs [llama.cpp](https://github.com/ggml-org/llama.cpp) inside a Docker container on a Linux machine with an NVIDIA GPU, loads models in GGUF format on demand, and exposes an OpenAI-compatible HTTP API plus a web chat UI on port 8081.

## What you get

- A single `docker compose` service that serves several models from one endpoint.
- Models are loaded into VRAM on the first request and unloaded after 10 minutes of inactivity, so only one model occupies the GPU at a time.
- Per-model settings (context size, temperature) in one file, `models.ini`.

## Requirements

| Component | Requirement |
|---|---|
| OS | Debian-based Linux (tested on Kali Linux) |
| GPU | NVIDIA GPU with enough VRAM for the chosen model (tested on an RTX 3060, 12 GB) |
| NVIDIA driver | A version that reports `CUDA Version: 12.8` or newer in `nvidia-smi` |
| Docker | Docker Engine and Docker Compose |
| NVIDIA Container Toolkit | Lets Docker containers use the GPU |
| Disk space | About 15 GB for the Docker image, plus the size of the models (5-9 GB each) |

Tested with driver 595.80 (CUDA 13.2) and Docker 28.5.2.

## Repository layout

```
.
├── docker-compose.yml   # How to start the llama.cpp container
├── models.ini           # Model presets (path, context size, temperature)
├── gguf/                # Model files (not tracked by git, you download them)
└── README.md
```

## Setup

### 1. Install Docker

```bash
sudo apt update
sudo apt install docker.io docker-compose
```

```bash
sudo systemctl enable --now docker
```

Allow your user to talk to the Docker daemon without `sudo`:

```bash
sudo usermod -aG docker $USER
```

Log out and back in, or run `newgrp docker` in the current terminal, so the new group is active.

### 2. Install the NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

```bash
sudo apt update && sudo apt install -y nvidia-container-toolkit
```

Register the NVIDIA runtime in Docker and restart it:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 3. Check the NVIDIA driver

```bash
nvidia-smi
```

The `CUDA Version` shown in the top-right corner must be 12.8 or newer, because the `full-cuda` image is built against CUDA 12.8+. If it is older, install a newer driver from the [NVIDIA download page](https://www.nvidia.com/en-us/drivers/) and reboot. Drivers installed with NVIDIA's `.run` installer are outside the package manager and may need to be reinstalled after a kernel upgrade.

### 4. Get the project and the models

```bash
git clone <this-repository-url> local-ai
cd local-ai
mkdir -p gguf
```

Download the models with the Hugging Face CLI (`hf`):

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install -U huggingface_hub
```

```bash
hf download unsloth/gemma-4-E4B-it-GGUF --include "gemma-4-E4B-it-UD-Q4_K_XL.gguf" --local-dir ./gguf
```

```bash
hf download bartowski/Dolphin3.0-Llama3.1-8B-GGUF --include "Dolphin3.0-Llama3.1-8B-Q8_0.gguf" --local-dir ./gguf
```

`--include` downloads only the requested quantization, since each repository holds many variants of the same model.

The file names must match the `model =` lines in `models.ini` exactly.

### 5. Start the server

```bash
docker compose up -d
```

If your system only has Compose v1, use `docker-compose up -d` instead. The first start pulls the Docker image, which can take a while.

### 6. Verify

```bash
docker logs llama-server
```

The output should end with a line saying the router server is listening on port 8081.

```bash
curl http://localhost:8081/v1/models
```

The response lists the presets defined in `models.ini`. Open `http://localhost:8081` in a browser for the chat UI, or call the API with one of the listed ids:

```bash
curl http://localhost:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "<id-from-/v1/models>", "messages": [{"role": "user", "content": "Hello"}]}'
```

## Configuration

### `docker-compose.yml`

| Setting | Meaning |
|---|---|
| `image: ghcr.io/ggml-org/llama.cpp:full-cuda` | llama.cpp built with CUDA support |
| `ports: "127.0.0.1:8081:8081"` | Publishes the API on the local machine only. Use `"8081:8081"` to reach it from the LAN, ideally behind a firewall |
| `./gguf:/models` | Mounts the model folder into the container |
| `--models-preset /config.ini` | Loads the presets from `models.ini` |
| `--models-autoload` | Loads a model when the first request for it arrives |
| `--models-max 1` | Keeps at most one model in VRAM |
| `--sleep-idle-seconds 600` | Unloads the model after 10 minutes without requests |
| `--host 0.0.0.0` | Listens on all interfaces inside the container. A server bound to `127.0.0.1` inside a container is not reachable through a published port |
| `capabilities: [gpu]` | Gives the container access to the GPU |

### `models.ini`

| Parameter | Meaning |
|---|---|
| `gpu-layers = 999` | Offloads as many layers as possible to the GPU |
| `flash-attn = on` | Enables Flash Attention, which lowers VRAM use |
| `cache-type-k/v = q4_0` | Stores the KV cache (the memory of the conversation) in 4 bits to save VRAM |
| `c = 32768` | Context size in tokens. Larger values need more VRAM |
| `temp = 0.8` | Sampling temperature. Lower is more deterministic |
| `fit = on` | Spills layers that do not fit in VRAM into system RAM |

To add a model, put the `.gguf` file in `gguf/` and add a section to `models.ini` with its `model =` path.

## Daily use

```bash
docker compose up -d
```

```bash
docker compose down
```

```bash
docker compose restart
```

```bash
docker logs -f llama-server
```

```bash
watch -n 1 nvidia-smi
```

Restart the container after every change to `models.ini`.

## Troubleshooting

| Message or symptom | Cause | Fix |
|---|---|---|
| `Unit docker.service not found` | Only the Docker client is installed | `sudo apt install docker.io` |
| `permission denied ... /var/run/docker.sock` | The user is not in the `docker` group | `sudo usermod -aG docker $USER`, then `newgrp docker` |
| `could not select device driver "nvidia" with capabilities: [[gpu]]` | NVIDIA Container Toolkit missing or not registered | Install it and run `sudo nvidia-ctk runtime configure --runtime=docker` |
| `unsatisfied condition: cuda>=12.8` | The NVIDIA driver is too old for the image | Install a newer driver |
| `request (N tokens) exceeds the available context size` | `c` in `models.ini` is too small | Raise `c`, then restart the container |
| The model does not load | The file name in `models.ini` does not match the file in `gguf/` | Compare with `ls gguf/` |

## Notes

- With 12 GB of VRAM, a context of 32768 tokens is about the practical maximum for the models above. Higher values cause out-of-memory errors; watch `nvidia-smi` while testing.
- The model has no memory between chats. Every new conversation starts from scratch.

## License

MIT, see [LICENSE](LICENSE).
