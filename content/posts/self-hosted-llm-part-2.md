---
title: "Adventures on Self-Hosting an LLM: Part 2 (Wake-On-Lan & Shutdown on idle)"
date: 2026-07-11T18:09:47+02:00
slug: "self-hosted-llm-part-2"
tags:
    - homelab
    - llm
    - self-hosted-llm-adventure
---

Moving on from [Part 1]({{< ref "self-hosted-llm-part-1" >}}), we have 3 tasks to achieve in this part:

1. Enable Wake-On-Lan for the backend host
2. Enable vLLM to run on boot
3. Auto-shutdown of backend host on idle timeout

## Wake-On-Lan (WoL) Setup

My main reference for this step is the [Debian Wiki on WoL](https://wiki.debian.org/WakeOnLan).

First thing I did is enable WoL via the BIOS. I have an ASRock motherboard and was able to find the setting under `Advanced Mode -> Advanced -> ACPI Configuration -> PCIE Devices Power On`. I set it to enabled then saved the changes and rebooted.

The next step is to enable it via software in Debian. My backend has 2 NICs so I ran `ip a` to see which is my main one (`enp3s0` in this case). I edited `/etc/network/interfaces` with:

```
# The primary network interface
iface enp3s0 inet dhcp
         ethernet-wol g  # <-- Add this to your primary NIC
```

then rebooted the system. I quickly verified that the setting survives a reboot by:

```bash
sudo ethtool enp3s0 | grep Wake-on
```

which returned

```bash
...snip...
Wake-on: g
...snip...
```

verifying that WoL is enabled. Last step is to test whether the backend will wake up by running the following on another machine:

```bash
wakeonlan <BACKEND MAC ADDR>
```

which promptly turned the backend on.

## vLLM On Boot

For this part, I took advantage of systemd to manage vLLM and run it on boot. First, I created a script (`/usr/local/bin/start-vllm.sh`) that contains the command to start vLLM as a docker container:

```bash
docker run --rm --runtime nvidia --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    --env "HF_TOKEN=$HF_TOKEN" \
    -p 8000:8000 \
    --ipc=host \
    vllm/vllm-openai:v0.24.0 \
    --model Qwen/Qwen3-8B-AWQ \
    --gpu-memory-utilization 0.90 \
    --max-model-len 4096 \
    --quantization awq \
    --enforce-eager
```

The command above should be familiar as it is almost the same as what I used in Part 1 except the model. Next up, I created `/etc/systemd/system/vllm.service`:

```bash
[Unit]
Description=vLLM OpenAI server
After=network-online.target docker.service
Requires=network-online.target docker.service

[Service]
ExecStart=/usr/local/bin/start-vllm.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

and then `sudo systemctl enable --now vllm.service` to register the service and start it. I verified the service with `sudo systemctl status vllm.service`.

## Auto Shutdown on Idle Timeout

For the last step, I wrote a bash script that runs on an interval (systemd timer) that checks whether vLLM is busy. If vLLM has been idle (no traffic) for the last 30 minutes, we send a shutdown command to Debian to turn off the backend. Now, the question is: How does the script know vLLM is idle or busy?

vLLM exposes a [metrics endpoint](https://docs.vllm.ai/en/stable/design/metrics/) which gives us information on various aspects of vLLM. The first metric I considered for the idle check is `num_requests_*` (running, waiting and swapped). This is the direct answer to the question, if there are running or pending requests, that means vLLM is busy. However, I realized midway through writing the bash script that `num_requests_*` is a gauge meaning, the value is either > 0 if vLLM is busy or 0 if idle. Since we are running the script on an interval, how do we avoid the scenario wherein the script scrapes the metrics in between requests, giving the false assumption that the server is idle?


To combat the edge case above, I decided to use `prompt_tokens_total` instead, which is a counter. We simply save the current value of `prompt_tokens_total` at every interval and if the value hasn't changed after 30 minutes then that means vLLM is idle and we can safely shut down. The only caveat here is that `prompt_tokens_total` doesn't handle long stream responses. If vLLM streams a response > 30 minutes, the script will consider this idle and prompt a shutdown. For my current use case, this is fine. An improvement would be to combine `prompt_tokens_total` and `num_requests_*` but that's for my future self to worry about.

```bash
#!/usr/bin/env bash
set -euxo pipefail

: "${IDLE_TIMEOUT:=1800}" "${METRICS_URL:=http://localhost:8000/metrics}"

TIME_FILE=/run/vllm-active-time
TOTAL_TOKENS_FILE=/run/vllm-total-tokens
NOW=$(date +%s)

if [ ! -f "$TIME_FILE" ]; then
    echo "$NOW" > "$TIME_FILE";
fi

if [ ! -f "$TOTAL_TOKENS_FILE" ]; then
    echo 0 > "$TOTAL_TOKENS_FILE";
fi

CUR_TOKENS=$(curl -s $METRICS_URL \
        | awk '/^vllm:prompt_tokens_total/ {print $2}' \
        || echo -1)

PREV_TOKENS=$(cat "$TOTAL_TOKENS_FILE")

if (( $(echo "$CUR_TOKENS > $PREV_TOKENS" | bc -l) )); then
    echo "$NOW" > "$TIME_FILE";
    echo "$CUR_TOKENS" > "$TOTAL_TOKENS_FILE";
    exit 0;
fi

LAST_ACTVE=$(cat "$TIME_FILE" 2>/dev/null || echo "$NOW")
IDLE=$(( NOW - LAST_ACTVE ))

if [ "$IDLE" -ge "$IDLE_TIMEOUT" ]; then
        echo "idle ${IDLE}s ≥ ${IDLE_TIMEOUT}s → SHUTTING DOWN";
        systemctl poweroff;
fi
```

As mentioned earlier, we schedule the script using systemd timer. I created the service file `/etc/systemd/system/vllm-idle-check.service` with:

```bash
[Unit]
Description=vLLM OpenAI server idle checker

[Service]
Type=oneshot
ExecStart=/usr/local/bin/vllm-idle-check.sh
```

and the partner timer file `/etc/systemd/system/vllm-idle-check.timer`:

```bash
[Unit]
Description=vLLM OpenAI server idle checker timer
After=vllm.service
Requires=vllm.service

[Timer]
OnBootSec=5min
OnUnitActiveSec=1min

[Install]
WantedBy=multi-user.target
```

Same with the vLLM service, I registered and started with timer with `sudo systemctl enable --now vllm-idle-check.timer` and verified with `sudo systemctl status vllm-idle-check.timer`.

I verified end-to-end by waking the backend using WoL, queried vLLM a couple of times while looking at the idle script log. Leaving vLLM idle for 30 minutes successfully triggered the shutdown branch and turned off the backend.

A valid question at this point is: How will my LLM GUI interface handle this setup? How will it know that it needs to perform a WoL first before sending the query? We'll answer that in part 3.
