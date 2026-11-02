---
layout: post
title: "Moving Homepage from Docker to k3s: what the dashboard loses and how to get it back"
date: 2026-11-02
categories: [infrastructure]
tags: [homepage, kubernetes, k3s, prometheus, snmp]
---

[Homepage](https://gethomepage.dev/) is the dashboard I use for my homelab. Originally I used a read-only Docker socket to get stats but after the recent k3s migration, the stats were no longer available. Most of the data moved to a new Grafana dashboard but after a while, I started to miss the homey feel of the old dashboard with at-a-glance stats. Here I will document the steps I took to recreate it.

# Status and stats: Kubernetes mode

Homepage has a [Kubernetes mode](https://gethomepage.dev/configs/kubernetes/) that reads pods instead of containers. It needs a ServiceAccount (the identity a pod uses to call the Kubernetes API) that can read pods, and the metrics API for CPU and memory. Get and list permissions are enough:

```yaml
kind: ClusterRole
rules:
  - apiGroups: [""]
    resources: [namespaces, pods, nodes]
    verbs: [get, list]
  - apiGroups: [metrics.k8s.io]
    resources: [nodes, pods]
    verbs: [get, list]
```

In `kubernetes.yaml` I set `mode: cluster`, so Homepage uses its own pod's ServiceAccount. I also set `ingress: false` - Homepage can build tiles by itself from annotations on the cluster's routes, but they would come in an unpredictable order, so I decided I do not need this feature at all and disabled it. I'm really confused why it's enabled by default to begin with.

Each tile names its pods. The simplest tile names an app and its namespace:

```yaml
- Minecraft:
    app: minecraft
    namespace: games
```

`app` matches pods with the standard label `app.kubernetes.io/name=minecraft`. For pods labelled some other way, or several apps on one tile, `podSelector` takes any label selector:

```yaml
- Immich:
    app: immich
    namespace: immich
    podSelector: app in (immich-server, immich-machine-learning, database, redis)
```

A tile can match many pods: Immich's four show as one card, with CPU and memory summed over all of them.

**Nothing showed up at first.** For the Immich tile I had only set `podSelector`, which looked like enough. In v2.6.0, a tile shows Kubernetes status only when it also has `app:`, even if the `podSelector` already picks the pods. No error, no log line, just an empty tile.

## "Partial" everywhere

With status working, most tiles said **partial**, including bots that run a single pod. That surprised me, because everything was clearly up.

Homepage counts a pod as fine when it's Running or Succeeded (finished cleanly), and says `partial` when only some of a tile's pods are. The rest were dead pods: 253 of them. k3s shuts the node down gracefully on a reboot: the kubelet, the agent that runs pods on a node, stops every pod first. Each pod stopped that way stays behind as Failed or Succeeded, and its Deployment (what keeps an app's pod running) starts a fresh one on boot. The Failed ones turned the tiles partial.

**My first fix only worked by luck.** Kubernetes deletes finished pods only once there are more than `terminated-pod-gc-threshold` of them, 12,500 by default, which a homelab never reaches. So I set it to 20 with a k3s server flag:

```
--kube-controller-manager-arg=terminated-pod-gc-threshold=20
```

The count dropped from 253 to 20 in about 30 seconds, and every tile said running. But the cleanup keeps the 20 newest finished pods, whatever they are. That day the newest happened to be Job pods (one-off tasks such as backups), so the old app pods went first. After the next reboot about 60 pods would die at once, Jobs keep only a handful of finished pods around, and a dozen dead app pods would have stayed.

**The fix: delete only what the shutdown stopped.** The kubelet marks each pod it stops for a shutdown with a `DisruptionTarget` condition, a note in the pod's status, saying "Pod was terminated in response to imminent node shutdown." A small systemd unit runs this once after k3s starts:

```bash
until k3s kubectl get --raw=/readyz >/dev/null 2>&1; do sleep 5; done
k3s kubectl get pods --all-namespaces --output json \
  | jq -r '.items[]
      | select(any(.status.conditions[]?; .type == "DisruptionTarget"
          and .message == "Pod was terminated in response to imminent node shutdown."))
      | "\(.metadata.namespace) \(.metadata.name)"' \
  | while read -r namespace pod; do
      k3s kubectl delete pod --namespace "$namespace" "$pod" --wait=false
    done
```

Matching the message, not just the condition, keeps out pods the kubelet evicts when memory runs low: they get the same condition with a different message. Finished Job pods have no such condition, so they stay until their Job's own cleanup.

My first version matched the pod's top-level status message instead. The reboot that tested it deleted 17 dead pods and missed 30: a pod that exits cleanly ends up Succeeded with an empty status message, and only the condition keeps the shutdown note. Matching the condition cleared those 30 as well. On the next reboot, the final version deleted all 47 pods the shutdown stopped, 34 seconds after boot, and left the Job pods alone.

# Health: Docker's "healthy" has no counterpart

This is one gap I could not bridge and I admit - Docker did better here. Docker shows healthy or unhealthy from the container's health check. In Kubernetes mode, Homepage reads only the pod's phase: Pending, Running, Succeeded or Failed. A pod failing its readiness probe, the check Kubernetes runs to decide whether a pod can take traffic, still says running.

As a workaround I used [`siteMonitor`](https://gethomepage.dev/configs/services/#site-monitor) on each tile (where it made sense at least). It calls a URL and shows the response time, or an error. I pointed it at the same path as the app's readiness probe, so the dashboard and Kubernetes agree on what "up" means:

```yaml
- Immich:
    siteMonitor: http://immich-server.immich:2283/api/server/ping
```

# Widgets behind NetworkPolicies

On Docker, Homepage joined two shared networks, the proxy's and one just for widgets, and every app it showed had to join one of them. In the cluster that job falls to NetworkPolicies: each app's policy admits only the ingress controller, so a widget or `siteMonitor` times out until its target lets Homepage in.

I added the rule on the target's side, in that app's own policy, on the port Homepage checks:

```yaml
ingress:
  - from:
      - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: homepage } }
        podSelector: { matchLabels: { app: homepage } }
    ports: [{ port: 2283 }]
```

It's more YAML than one wide rule on Homepage's side, but each app's policy then lists everything that can reach it.

# Widgets without admin accounts

A few widgets wanted more access than I wanted to give to a dashboard. For two of them, SNMP was the way around: the protocol routers and NASes use to report their numbers.

| Widget | Needs | What I did |
|---|---|---|
| Grafana | a Grafana server admin (it calls `/api/admin/stats`) | dropped it: four counts weren't worth an admin login |
| Synology (`diskstation`) | an account in DSM's Administrators group | SNMP into Prometheus, then `prometheusmetric` |
| Router | none exists | the same: SNMP into Prometheus |

My router was already in Prometheus through [snmp_exporter](https://github.com/prometheus/snmp_exporter), which turns SNMP into Prometheus metrics, with a Grafana dashboard on top. So I took the dashboard's queries and put them in a [`prometheusmetric`](https://gethomepage.dev/widgets/services/prometheusmetric/) widget: WAN down and up, CPU and memory. For the NAS I made a read-only SNMPv3 user in DSM and scraped it with snmp_exporter's `synology` module. Its widget shows free space, CPU, memory and temperature:

```yaml
widget:
  type: prometheusmetric
  url: http://prometheus.monitoring:9090
  metrics:
    - label: Temp
      query: temperature{instance="nas"}
      format: { type: number, suffix: "°C" }
```

I like this better than the native widgets anyway: the numbers on the dashboard are the same ones Grafana graphs.

**SNMP is read-only.** An SNMP user can be allowed to change settings as well as read them, so I went to check on the permissions on the relevant files. On the router I configured it by hand, so nothing to check, but DSM was enabled through the UI, so it needed a check. Thankfully the answer was simple.

```
$ sudo grep -E '^(rouser|rwuser|rocommunity|rwcommunity)' /etc/snmp/snmpd.conf
rocommunity syno localhost
rouser prometheus
```

`rouser` means read-only. The `syno` community is DSM's own and only answers the NAS itself.

# Setup

Homepage v2.6.0 on one node of k3s v1.35.8+k3s1 (NixOS), as of 2026-11-02.

# Closing thoughts

In general, I knew moving to k3s would be a bumpy ride, as it really only starts making sense in a multi-node environment, and Docker really had all I needed, but I wanted to learn, and just for that all these bumps are worth it. I hope you find these useful; they took me a while to figure out.

I can safely say it was fun tinkering with this.