---
title: "Adventures on Self-Hosting an LLM: Part 0"
date: 2026-06-29T19:17:21+02:00
slug: "self-hosted-llm-part-0"
tags:
  - homelab
  - llm
---

I'm currently trying to understand how LLMs actually work, so I did what any reasonable person would do, self-hosting one. I'll be documenting my adventure in a series of posts, kicking it off with this one.

My rough idea for the setup looks something like:

```
                    +------------------+
                    |Kubernetes Cluster|
                    |                  |
User ------------>  | +--------------+ |
                    | | LLM Gateway  | |
                    | +--------------+ |
                    +--------|---------+
                             |
                             | LLM Queries /
                             | Magic Packet (WoL)
                             |
                    +--------v---------+
                    | Desktop with GPU |
                    |  (Wake-On-LAN)   |
                    |                  |
                    | +--------------+ |
                    | |Inference     | |
                    | |Engine /      | |
                    | |LLM           | |
                    | +--------------+ |
                    +------------------+
```

One core constraint I'm imposing on the setup is that this setup should not cost me an arm and a leg on electricity. So, I've split the setup into two: Gateway inside the cluster (always on) and the Engine/LLM on the desktop (on-demand).

The immediate problem with the setup is that if a request comes in and the Desktop is off, how would that be served? This is where Wake-On-LAN comes into play. If the backend is off and a user starts a query, the gateway will turn on the backend using Wake-On-LAN and systemd should take care of starting the inference engine.

I hope to explain (read: figure out) the nitty-gritty details as we proceed. But for now, this is the overall plan and see you in the next part!
