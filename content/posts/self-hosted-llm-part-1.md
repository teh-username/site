---
title: "Adventures on Self-Hosting an LLM: Part 1 (Engine & Model Setup)"
date: 2026-07-04T20:08:22+02:00
slug: "self-hosted-llm-part-1"
tags:
    - homelab
    - llm
    - self-hosted-llm-adventure
---

Picking up from [Part 0]({{< ref "self-hosted-llm-part-0" >}}), I'll setup my Desktop to be able to host an inference engine and an LLM model. I'll perform a sample query to the LLM after to verify.

I already have an existing OS on my Desktop so I'll be dual-booting a fresh Debian (13.5 to be exact) to keep things separate. As for the inference engine, I'll be using [vLLM](https://vllm.ai/) alongside my RTX 3070Ti GPU (not much VRAM I know, but we'll make it work).

So, first things first, let's install the dependencies.

## Docker

I'll be using Docker to run vLLM to simplify dependency management. As such, I've installed Docker from the `apt` repository as documented [here](https://docs.docker.com/engine/install/debian/#install-using-the-repository). I've also gone through some of the [post-install steps](https://docs.docker.com/engine/install/linux-postinstall), mainly to manage Docker as a non-root user and configuring Docker to start on boot.

## NVIDIA Container Toolkit Setup

### Toolkit Installation

Next up is installing the [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-container-toolkit) to support GPU access to the containerized vLLM. I followed the [installation guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) under the `apt` repository:

1. Pre-requisites

```shell
sudo apt-get update && sudo apt-get install -y --no-install-recommends \
   ca-certificates \
   curl \
   gnupg2
```

2. Configure repository

```shell
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

3. Update package list

```shell
sudo apt-get update
```

4. Install the toolkit

```shell
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.19.1-1
  sudo apt-get install -y \
      nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

After which I added the NVIDIA container runtime in Docker by running:

```shell
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### NVIDIA Driver Installation

I also needed to install the [NVIDIA Drivers for Debian](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/latest/debian.html#debiInstall). Otherwise, if I run the container toolkit verification immediately, I get the following error:

```
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: could not apply required modification to OCI specification: error modifying OCI spec: failed to create the automatic CDI modifier: failed to generate CDI spec for mode "auto": failed to construct device spec generators: failed to initialize NVML: ERROR_LIBRARY_NOT_FOUND
```

To install the NVIDIA Drivers, I just followed the link above and ran the following:

1. Pre-requisites

```shell
sudo apt install linux-headers-$(uname -r)
```

2. Install cuda-keyring package and update package list

```shell
wget https://developer.download.nvidia.com/compute/cuda/repos/debian13/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
```

3. Install Drivers (Compute-only / Headless)

```shell
sudo apt -V install nvidia-driver-cuda nvidia-kernel-dkms
```

4. Reboot

```shell
sudo reboot
```

Once the system is back up, I verified the driver installation by running:

```shell
nvidia-smi
```

Which outputs:

```shell
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 610.43.02              KMD Version: 610.43.02     CUDA UMD Version: 13.3     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3070 Ti     On  |   00000000:01:00.0  On |                  N/A |
|  0%   33C    P8             12W /  290W |      17MiB /   8192MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

Running the container toolkit verification:

```shell
docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

Gives me the same output as `nvidia-smi`.


## Running vLLM

Let's now put everything together by running vLLM via Docker:

```shell
docker run --runtime nvidia --gpus all \
      -v ~/.cache/huggingface:/root/.cache/huggingface \
      --env "HF_TOKEN=$HF_TOKEN" \
      -p 8000:8000 \
      --ipc=host \
      vllm/vllm-openai:v0.24.0 \
      --model Qwen/Qwen3-0.6B \
      --max-model-len 4096 \
      --gpu-memory-utilization 0.90 \
      --enforce-eager \
      --default-chat-template-kwargs '{"enable_thinking": false}'
```

The combination of `--max-model-len`, `--gpu-memory-utilization` and `--enforce-eager` are required to be able to fit the model on the VRAM of my GPU. The `--default-chat-template-kwargs` just forces Qwen to disable "thinking" which is enabled by default.

Opening another shell, I can then verify vLLM by running:

```shell
curl http://localhost:8000/v1/models
```

which gives me:

```shell
{
  "object": "list",
  "data": [
    {
      "id": "Qwen/Qwen3-0.6B",
      "object": "model",
      "owned_by": "vllm",
      "root": "Qwen/Qwen3-0.6B",
      "parent": null,
      "max_model_len": 4096,
...snip...
```

and what we came for:

```shell
curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "Qwen/Qwen3-0.6B",
        "messages": [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "What sound does a fox make?"}
        ]
    }' | jq
```

which returns

```shell
{
  "id": "chatcmpl-8b5da16aaf0ed845",
  "object": "chat.completion",
  "model": "Qwen/Qwen3-0.6B",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "A fox makes a sound like a \"mew\" when it's looking. It's a common and distinctive sound associated with foxes in many regions.",
        "refusal": null,
        "annotations": null,
        "audio": null,
        "function_call": null,
        "reasoning": null
      },
...snip...
```

I'm calling that a success. We'll tackle the Shutdown-On-Idle and Wake-On-Lan on the next one. See you then!
