---
title: Locally hosting Plausible with Cloudflare Tunnels
date: 2023-02-05 16:34:07
tags:
    - homelab
    - analytics
---

I've recently discovered [Plausible](https://plausible.io/) from a 2022 /r/selfhosted round-up [thread](https://www.reddit.com/r/selfhosted/comments/zwcns3/most_used_selfhosted_services_in_2022), casually looking for new stuff to try out. Coincidentally, I've been running my website analytics free for quite a while now. Sounds like a good excuse to try it out.

Plausible is, in their own words:

> Simple, open-source, lightweight (< 1 KB) and privacy-friendly web analytics alternative to Google Analytics.

While Plausible offers a "managed" version, I decided to spice things up and go with the self-hosted version. In this post, I'll walk you through my setup.

### Requirements

Before proceeding to the fun stuff, I'd like to mention some requirements I had that influenced how I approached the setup. These are:

- Local-only access to Plausible's dashboard
- No direct internet traffic to the VM hosting Plausible

These two criteria are somewhat related and can be boiled down to me being paranoid about exposing my homelab to the internet (and mostly lazy as well). I'd like to avoid waking up to see my homelab pwned because of a misconfiguration.

The next question then is, how? We need a publicly accessible compute to serve the script and receive events on page visits. If the local VM is not reachable, how can Plausible do its job? Enter [Cloudflare Tunnels](https://www.cloudflare.com/products/tunnel/).

With Cloudflare Tunnels, we can front a Cloudflare server towards the internet and only let specific traffic "tunnel" back to the local VM. This way, we offload the server hardening and general security to Cloudflare while enabling the local VM to be accessible as if it was exposed to the internet.

### The Plan

```
                  |
      Internet    |    Local Homelab
                  |
+-----------+     |
|  Website  |     |
+-----------+     |    +--------------------------------------+
      |           |    |                                      |
      |           |    |                      +---------+     |
      |           |    |           +--------> |  Nginx  |     |
      v           |    |           |          +---------+     |
+------------+    |    |    +-------------+        |          |
| Cloudflare |------------->| cloudflared |        |          |
+------------+    |    |    +-------------+        v          |
                  |    |                     +-----------+    |
                  |    |                     | Plausible |    |
                  |    |                     +-----------+    |
                  |    |                                      |
                  |    +--------------------------------------+
                  |
                  |
```

With the requirements in mind, the diagram above summarizes the setup. The public website will be communicating with Plausible via Cloudflare. A tunnel is set up between an internet-facing Cloudflare server and the VM hosting Plausible (inside the local network).

This way, we control what type of traffic we let through the tunnel and only allow local access to Plausible's dashboard.

#### Why a reverse proxy?

Before proceeding, you might be thinking, why complicate the setup with a reverse proxy in front of Plausible? Can't we just point `cloudflared` to Plausible?

To answer your question, yes, we can point `cloudflared` to Plausible, but, we'll expose Plausible's dashboard to the internet. This violates the first requirement of allowing local-only access to Plausible.

The main reason for the reverse proxy is that we only want to expose 2 URLs that will be served by Plausible:

1. `js/script.js` - This endpoint will serve the analytics script
2. `api/event` - This endpoint receives the data gathered by the analytics script

Our reverse proxy will only respond to these 2 endpoints and drop everything else. This fulfills our first requirement while allowing Plausible to perform its job unhindered.

### Setup

Prerequisites for this setup are: compute on your local network and a domain managed by Cloudflare.

#### Plausible

_Note: Most (if not all) of the information in this section are from Plausible's [self-hosting documentation](https://plausible.io/docs/self-hosting)._

Plausible's self-hosted version is only offered under Docker. So before anything else, make sure your server has Docker installed. [Install instructions](https://docs.docker.com/get-docker/)

In summary, you need to do the following:

1. Get the Code

```bash
git clone https://github.com/plausible/hosting
cd hosting
```

2. Set Configuration

There are only two keys you need to fill up for the config file (`plausible-conf.env`):
- `SECRET_KEY_BASE` - Plausible suggests generating the secret key via:

```bash
openssl rand -base64 64 | tr -d '\n' ; echo
```

- `BASE_URL` - This would be the domain managed by Cloudflare that will make our Plausible instance accessible via the tunnel.

3. Start the App

```bash
docker-compose up -d
```

Plausible should now be accessible at `http://{COMPUTE_IP}:8000`

You can set up a website inside Plausible at this point. Make sure to note down the JavaScript snippet that Plausible will generate for you. We'll need that later.

#### Nginx

The next step is to install [Nginx](https://www.nginx.com/). Please refer to your OS of choice on how to install it. If you are using Debian/Ubuntu:

```bash
sudo apt update
sudo apt install nginx
```

The following config enables the bare minimum functionality for our setup:

```
server {
    listen      80;
    listen      [::]:80;
    server_name YOUR.DOMAIN.HERE;
    root        /var/www/your/root/here;

    # analytics script
    location /js/script.js {
        proxy_pass            http://127.0.0.1:8000/js/script.js;
    }

    # API call
    location /api/event {
        proxy_pass            http://127.0.0.1:8000/api/event;
    }
}
```

I recommend [NGINXConfig by Digital Ocean](https://www.digitalocean.com/community/tools/nginx) to help generate the complete set of config files for Nginx.

You can verify your setup by:

```bash
curl --header "Host: YOUR.DOMAIN.HERE" http://IP.OF.PLAUSIBLE/js/script.js
```

It should return a blob of JavaScript that contains the analytics code.

#### Tunnel

The last step would be to configure your Cloudflare tunnel. I'd recommend using the web interface in this case since the setup isn't that complicated. For setup via CLI, see [documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/local/).

Using the web interface:

1. Open your Cloudflare dashboard > Click Zero Trust
2. Click Access > Tunnels
3. Create a tunnel
4. Follow the succeeding instructions on how to install `cloudflared`
5. Choose "Public Hostnames"
    * Make sure the public hostname is the same as the `BASE_URL` you used to configure Plausible
    * Under "Service" choose: Type = HTTP, URL = localhost:80
6. Save

You can verify that the tunnel is working by visiting `https://BASE_URL/js/script.js`. It should return the same JavaScript blob when we tested via `curl`.

Once you've verified that the `js/script.js` is publicly accessible, embed the JavaScript snippet Plausible generated a while ago within the `head` tag of the website. More info on how to do it [here](https://plausible.io/docs/plausible-script).

Finally, visit your website and marvel at the initial number in your Plausible dashboard!
