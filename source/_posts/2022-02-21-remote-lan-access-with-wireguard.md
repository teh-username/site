---
title: Remote LAN access with WireGuard
date: 2022-03-13 18:15:03
tags:
  - homelab
  - networking
---

In this episode, let's go over how to set up a simple but secure tunnel (read: VPN) to your local LAN (read: homelab) using [WireGuard](https://www.wireguard.com/). We'll be going with the VPS route so we don't have to expose any ports to the internet.

We'll be emulating the following setup:

{% asset_img wg_topo.jpg REEEEEE %}

The Cast:

* "Router" - The machine that will serve as the gateway (inwards) to your LAN
* "Server" - The machine with a publicly accessible IP that all clients will connect to. Also known as a "Bounce Server"
* "Client" - You, trying to connect to your LAN remotely somewhere

## Boring (get it?) Setup

_Note: All the machines here are Ubuntu-based. Adjust the setup accordingly to your distro of choice._

### "Server" and "Router"

For "Server" and "Router" perform the following:

```bash
sudo apt update && sudo apt upgrade
sudo apt install wireguard
wg genkey | tee privatekey | wg pubkey > publickey
sudo sysctl net.ipv4.ip_forward=1
```
_Note: To persist IP Forwarding, edit `/etc/sysctl.conf` with `net.ipv4.ip_forward=1`_

For the "Server", create `/etc/wireguard/wg0.conf` with:

```ini
[Interface]
Address = 192.168.10.1/32
ListenPort = 51820
PrivateKey = <Server's Private Key>

# Router Peer
[Peer]
PublicKey = <Router's Public Key>
AllowedIPs = 192.168.10.0/24, 10.0.20.0/24
```

For the "Router", create `/etc/wireguard/wg0.conf` with:

```ini
[Interface]
Address = 192.168.10.3/32
PrivateKey = <Router's Private Key>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE

# Server
[Peer]
PublicKey = <Server's Public Key>
Endpoint = <Server's Public IP>:51820
AllowedIPs = 192.168.10.0/24
PersistentKeepalive = 25
```
_Note: Replace `ens18` with the appropriate interface_

Enable the interface by `wg-quick up wg0` and then check the status by `wg show`.

At this point, we can perform a quick sanity check. On my setup, running `mtr 10.0.20.1` on the "Server" yields:

```bash
                          Packets               Pings
 Host                   Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.10.3        0.0%     2   61.8  41.5  21.1  61.8  28.8
 2. 10.0.20.1           0.0%     2   48.3  33.0  17.6  48.3  21.7
```

### "Client"

On the "Client" perform the following:

```bash
sudo apt update && sudo apt upgrade
sudo apt install wireguard
wg genkey | tee privatekey | wg pubkey > publickey
```

Then create `/etc/wireguard/wg0.conf` with:

```ini
[Interface]
Address = 192.168.10.2/32
PrivateKey = <Client's Private Key>

[Peer]
PublicKey = <Server's Public Key>
Endpoint = <Server's Public IP>:51820
AllowedIPs = 10.0.20.0/24
PersistentKeepalive = 25
```
Enable the interface by `wg-quick up wg0` and then check the status by `wg show`. We also need to update the `wg0.conf` of "Server" with "Client" as a new peer.

Update "Server" with:

```ini
[Interface]
Address = 192.168.10.1/32
ListenPort = 51820
PrivateKey = <Server's Private Key>

# Router LAN
[Peer]
PublicKey = <Router's Public Key>
AllowedIPs = 192.168.10.0/24, 10.0.20.0/24

# Client
[Peer]
PublicKey = <Client's Public Key>
AllowedIPs = 192.168.10.2/32
```
_Note: Don't forget to restart the `wg0` interface by `wg-quick down wg0 && wg-quick up wg0`_

Now, running `mtr 10.0.20.1` on the "Client" yields:

```bash
                          Packets               Pings
 Host                   Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.10.1        0.0%     6   41.2  42.1  21.4  73.6  18.7
 2. 192.168.10.3        0.0%     5   64.8  72.0  41.8  88.1  19.3
 3. 10.0.20.1           0.0%     5   77.4  62.3  43.0  77.4  13.3
```

Which follows the "Client" -> "Server" -> "Router" flow that we want.

## Config digging

Most of the "routing" is dictated by the `[Peer] AllowedIPs` configuration. It governs what can go in and go out of the tunnel. For example, given the following setup:

Peer A

```ini
[Interface]
Address = 192.168.10.1/32
...snip...

[Peer]
PublicKey = <Peer B's public key>
AllowedIPs = 192.168.10.0/24
```

Peer B

```ini
[Interface]
Address = 192.168.10.2/32
...snip...

[Peer]
PublicKey = <Peer A's public key>
AllowedIPs = 192.168.20.0/24
```

If Peer A attempts to connect to (for example) `192.168.10.11`, since the address is within `192.168.10.0/24`, it will be allowed to "enter" on Peer A's side. When it comes out of Peer B's side, it will be dropped since it is not within `192.168.20.0/24`. It gets a bit confusing when a "netmasked" `[Interface] Address` comes into play. While `[Interface] Address` can also influence the "routing" (if netmasked), the "final decision" is always up to the `[Peer] AllowedIPs`. For example:

Peer A

```ini
[Interface]
Address = 192.168.10.1/24
...snip...

[Peer]
PublicKey = <Peer B's public key>
AllowedIPs = 192.168.20.0/24
```

If we try to connect to `192.168.10.11`, it will be "routed" to this interface because of `192.168.10.1/24`, but it will not be allowed inside the tunnel (dropped) since it is not within `192.168.20.0/24`.

To demonstrate further, using our setup:

* Update `[Peer] AllowedIPs` of "Router" from `AllowedIPs = 192.168.10.0/24` to `AllowedIPs = 192.168.10.2/32`

"Client" still can reach `10.0.20.1/24` albeit "Server" cannot anymore.

* Update `[Peer] AllowedIPs` of "Client" from `AllowedIPs = 10.0.20.0/24, 192.168.10.0/24` to `AllowedIPs = 10.0.20.0/24`

"Client" is still able to reach `10.0.20.1/24`, but `mtr` does not display the IPs of the host in the chain anymore (e.g.)

```bash
                                           Packets               Pings
 Host                                Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. (waiting for reply)
 2. (waiting for reply)
 3. 10.0.20.1                         0.0%   222   41.7  47.1  37.3 180.8  15.5
```

The reason is that when the hosts try to "reply" to `mtr`, the packets are dropped. After all, "Server" (`192.168.10.1`) and "Router" (`192.168.10.3`) are not within the new `[Peer] AllowedIPs` range of "Client".

Till next time!
