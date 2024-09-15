---
layout: post
title: "How I exposed my local server on the internet using Tailscale"
date: 2024-09-15
categories: infrastructure tailscale docker traefik vps https tls
---

# How I exposed my local server on the internet using Tailscale with a custom domain and https

## Disclaimer

Please for the love of god, do not use this setup for production use. My programs mostly have no users therefore I could get away with the limitations of latency/bandwidth/just plain reason.

Also my situation is highly specific so far simpler solutions exist to what essentially is a non-issue. Treat this as a proof of concept, rather than something to be taken seriously :D.

## The Problem

My AWS free trial is expiring that hosts the one non-static or serverless website I have. Now a sane person would either: (1) drop the project, (2) start paying $7.2/mo. to Amazon, or (3) switch hosts. I didn't want to drop the project, and it was definitely not worth the monthly fee, so I decided to switch to something I already had: my private server I have at home.

I would start the migration till I quickly find out that I do not have an external IP address. I also seem to have no means of getting one either.

Thinking outside the box for a sec I decided to go with a cheap local VPS for $2 a month to act as an external IP address. Idea is that if I can make a VPN tunnel, I can set up a reverse-proxy on the device and forward the requests within the VPN.

Simple, I install tailscale and then run...

```
wgengine.NewUserspaceEngine(tun "tailscale0") error: tstun.New("tailscale0"): CreateTUN("tailscale0") failed; /dev/net/tun does not exist
```

Oh... is module there?

```bash
$ sudo modprobe tun
# modprobe: FATAL: Module tun not found in directory /lib/modules/6.1.0
```

Well that throws a wrench in my plans. Time for plan B.

## Plan B

While [Plan O](#plan-o---just-expose-the-port-dummy) (allow users to access your machine directly) and [Plan A](#plan-a---just-use-a-vpn) (use a VPN) are clearly better, more secure, and require significantly less effort, they also require technical infrastructure that I simply cannot obtain or unwilling to pay for.

### The Idea

Here is the general idea I had before I put everything together.

[Tailscale](https://tailscale.com/) offers a service called [Funnel](https://tailscale.com/kb/1223/funnel), which allows users to expose their websites (specifically), to the internet. This comes with 2 issues, as (1) it only allows for HTTPS traffic, and (2) it does not allow you to pick the hostname (cannot use CNAME to redirect either as they take out a certificate for you which would cause TLS issues).

The first issue is not much a big deal for now, as I want to expose websites, but I would like to expose things like minecraft servers too in the future. For now it is good. The second issue is not really much of an issue either, however, hear me out, custom domain pretty :>.

Okay! We now have a potential way to expose our service to the internet, but we need to still set up the proxy. For that I will just use [traefik](https://doc.traefik.io/traefik/). Any other proxy works too, but for this example I will use traefik.

### Putting it together - Tailscale

To serve something via Tailscale funnel is quite easy! If you have tailscale installed, you can run:

```bash
$ sudo tailscale serve http://localhost:5000
# Available within your tailnet:
# 
# https://host.your-tailnet.ts.net/
# |-- proxy http://localhost:5000
```

Note: First time setup will prompt you with configuration to enable the features ([docs](https://tailscale.com/kb/1312/serve#setup)).

This allows us to access the service via https on the tailnet, which is useful, but not that useful as it requires client to be logged in. What I want is to make this publicly accessible which is as simple as running `funnel` instead:

```bash
$ sudo tailscale serve http://localhost:5000
# Available on the internet:
# 
# https://host.your-tailnet.ts.net/
# |-- proxy http://localhost:5000
```

### Putting it together - Tailscale Docker

Running serve from host machine is good and all, but we can do better - we can run Tailscale in a docker container together with the service. I personally host everything in Docker anyway, so this works out great.

In general, I was following [a guide by Tailscale](https://tailscale.com/blog/docker-tailscale-guide), and it worked wonders!

In short:
1. In [ACLs](https://login.tailscale.com/admin/acls/file) add `tagOwners` as described in the guide:

``` json
	"tagOwners": {
		"tag:container": ["autogroup:admin"],
	}
```

2. Again in ACLs update `nodeAttrs.target` of Funnel policy to include `tag:container`, as seen on [this comment](https://github.com/tailscale/tailscale/issues/11849#issuecomment-2211972964).

```json
	"nodeAttrs": [
		{
			// Funnel policy, which lets tailnet members control Funnel
			// for their own devices.
			// Learn more at https://tailscale.com/kb/1223/tailscale-funnel/
			"target": ["autogroup:member", "tag:container"],
			"attr":   ["funnel"],
		},
	],
```

3. Create `OAuth` client with scope `devices:write`, and tag `tag:container`. Feel free to give a `Description` too.

<img src="./2024-09-15-tailscale-funnel.assets/oauth-client.png" alt="Example of creation window" height="800" />

4. Put together a `docker-compose.yml` file:

```yml
# ./docker-compose.yml

services:
  tailscale:
    image: tailscale/tailscale:stable
    hostname: ws2 # this is the host that appears on the internet
    environment:
      - TS_AUTHKEY=<YOUR_KEY_GOES_HERE>
      - TS_EXTRA_ARGS=--advertise-tags=tag:container
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve.json
    volumes:
      - ./data/state:/var/lib/tailscale
      - ./config:/config
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin
      - sys_module
    restart: unless-stopped

  app:
    image: traefik/whoami
    network_mode: service:tailscale
    depends_on:
      - tailscale
```

```json
// ./config/serve.json
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:80"
        }
      }
    }
  },
  "AllowFunnel": {
    "${TS_CERT_DOMAIN}:443": true
  }
}

```

> Here the docker-compose file is almost 1:1 with the one in the guide, and the `serve.json` is taken from the guide as well.
>
> The important bits for you to know is the application will work in the network of the tailscale container (can also work vice-versa), meaning even though tailscale doesn't expose port 80, the `app` does, and therefore a request can be proxied to `http://localhost:80`. Do note that the proxy ONLY works to localhost, so this is the only way of doing this.
>
> If you have more than 1 app you want to expose, you can set up another reverse proxy, or just repeat this pattern for every service you wish to expose.

Running `docker compose up -d` allowed me to access this container from the internet on the url: https://ws2.your-tailnet.ts.net/. the `ws2` bit comes from the container hostname. Unfortunately this means this will not load balance well, but this is for applications with no users anyway so who cares 😆.

Finally, wait a few minutes if you do not see your website appear initially. However, if after some time you see that the website still is not available, here are some troubleshooting tips:

1. Check if the host is up [on the dashboard](https://login.tailscale.com/admin/machines). You should see something like the following:
![Image showing host being up](./2024-09-15-tailscale-funnel.assets/host%20is%20up.png)

2. Check if you can access the url from a machine on tailnet. DNS propagates quicker internally so good way to check. If its not working from within a tailnet, something has gone wrong.

3. Give more permissions to OAuth token. Personally I had issues with DNS until I gave `dns:write` OAuth token scope, after which all started working, however, it continued working after i returned to original token, so your mileage may vary.

### Putting it together - Proxy Docker

Proxying requests should be somewhat straight forward, any proxy works, but I will show you my solution for completeness. This also requests uses Let's Encrypt to create certificates.

```yml
# docker-compose.yml

services:
  reverse-proxy:
    image: traefik:v3.1
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./traefik:/etc/traefik
      - ./letsencrypt:/letsencrypt
```

```yml
# ./traefik/traefik.yml

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

accessLog: {}

providers:
  file:
    directory: /etc/traefik/providers
    watch: true

tls:
  options:
    default:
      sniStrict: true

certificatesResolvers:
  main:
    acme:
      email: YOUR_HTTPS_EMAIL
      storage: /etc/traefik/acme.json
      httpchallenge:
        entrypoint: web
```

```yml
# ./traefik/providers/my-machine.yml
#   The name of file can be anything, just needs to be in the folder.

http:
  routers:
    ws2: # <- Dynamic name
      entrypoints:
        - websecure
      service: testing
      rule: Host(`ws2.gedas.dev`)
      tls:
        certResolver: main

  services:
    ws2: # <- Dynamic name
      loadBalancer:
        servers:
          - url: https://ws2.your-tailnet.ts.net/

```

With this configuration, that's it! Accessing https://ws2.gedas.dev/ will use `traefik` proxy to make a request to https://ws2.your-tailnet.ts.net/, which will use tailscale to funnel request to the appropriate docker container.

## Alternatives

### Plan O - Just expose the port.. dummy

When proxying via Cloudflare, its safe-ish to just expose the server you have at home and allow only [CloudFlare IPs](https://www.cloudflare.com/ips/) to access your machine via firewall rules, however I don't really have the option of having an external IP (thanks ISP).

### Plan A - Just use a VPN

What I am doing here is effectively a VPN, but just significantly less private. This was my first idea too, however the VPS provider I use does not permit access to `/dev/net/tun`, so I was out of luck setting up Tailscale.

A different VPS, or a different VPN may have solved this issue, but I didn't try those approaches.

### Plan O-1 Just use the VPS to host the whole server

... fair, I do have no users, my cheap little-to-no resource VPS would have worked regardless.

However, I did want to have MY data on MY server. If I have a project that blows up I will obviously just use this approach as eventually my own internet bandwidth would start to suffer, but for the projects I currently host - this is nothing.

### Plan B-0 Just use the URL provided by the funnel

... also fair, but I wanted my pretty domain :>

## Closing thoughts

If what you read sounds insane, its because it is. You should not do this. But I had fun doing it, and I feel like I should get back to documenting my findings more often, so here we are.

From a security perspective, if I didn't want something exposed on the internet, I would not expose it in the first place, and if I do, I would see if a funnel is a good idea in the first place. People can just install a tailscale client after all, its a great product!