---
title: "Why Your Kubernetes Rolling Updates Cause 502 Errors (And How to Fix Them)"
description: "A practical SRE guide to eliminating 502s during Kubernetes deployments — covering readiness probes, startup probes, rolling update strategy, preStop hooks, and graceful shutdown."
pubDate: 2026-06-20
author: "GoLiveApp"
tags: ["kubernetes", "devops", "sre", "502", "rolling-update", "k8s"]
---

You pushed a deployment. The rolling update kicked off. And then your phone buzzed — 502s. Clients were getting errors for 30, 60, sometimes 90 seconds before things stabilized.

This is one of the most common production war stories in the Kubernetes world, and almost every team hits it at least once. I hit it last month during a routine app upgrade, and the root cause turned out to be four things happening together:

1. No **startup probe** configured
2. No **readiness probe** configured
3. **Rolling update strategy** not tuned
4. No **preStop hook** or **terminationGracePeriodSeconds** set

This post is the guide I wish I'd had. By the end, you'll understand exactly why 502s happen and have copy-pasteable configs to prevent them.

---

## Why 502s Happen During Rolling Updates

When Kubernetes rolls out a new version of your app, it spins up new pods and terminates old ones. The 502 "Bad Gateway" error appears when traffic is routed to a pod that:

- **Isn't ready yet** — the new pod is still starting up (JVM warming up, DB connections establishing, etc.)
- **Is being killed** — the old pod received a SIGTERM and is shutting down, but Kubernetes is still sending it traffic

Both scenarios are entirely avoidable with the right configuration.

---

## Fix 1: Startup Probe

### The Problem

Without a startup probe, Kubernetes uses the readiness/liveness probe from the very first second. If your app takes 20 seconds to boot, your liveness probe will kill it before it even has a chance to start — causing a crash loop.

Startup probes tell Kubernetes: *"give this app time to initialize before you start checking its health."*

### The Fix

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 5
```

This gives the app up to `30 × 5 = 150 seconds` to start before Kubernetes declares it failed. Once the startup probe succeeds once, it hands off to the liveness probe.

**Rule of thumb:** Set `failureThreshold × periodSeconds` to at least 2× your worst-case cold start time.

---

## Fix 2: Readiness Probe

### The Problem

This is the big one for 502s. Without a readiness probe, Kubernetes adds a new pod to the service's endpoint list the moment it's `Running` — even if the app isn't actually ready to serve traffic. If a request hits the pod before the app is up, you get a 502.

### The Fix

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
  successThreshold: 1
```

**Key distinction:**
- `/healthz` (liveness) — "Is the app alive?" If this fails, restart the pod.
- `/ready` (readiness) — "Is the app ready to accept traffic?" If this fails, remove from load balancer rotation.

Your `/ready` endpoint should check real dependencies: database connection, cache warm-up, config loaded. Don't just return 200.

```go
// Example: Go readiness endpoint
func readyHandler(w http.ResponseWriter, r *http.Request) {
    if !db.Ping() || !cache.IsReady() {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
}
```

---

## Fix 3: Rolling Update Strategy

### The Problem

The default `maxUnavailable: 25%` and `maxSurge: 25%` settings are conservative but not always right. The issue is that with a small replica count (say, 2 pods), 25% rounds down — meaning 0 pods can be unavailable, so the rollout stalls, or worse, all pods cycle at once.

### The Fix

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # Never take a pod down before a new one is ready
      maxSurge: 1            # Only spin up 1 extra pod at a time
```

With `maxUnavailable: 0`, Kubernetes will never remove an old pod until a new pod has passed its readiness probe. This is the safest setting for production.

**For larger deployments**, you can tune this:

```yaml
rollingUpdate:
  maxUnavailable: 1    # Allow 1 pod to be down at a time
  maxSurge: 2          # Spin up 2 extra pods to speed up rollout
```

**Tip:** If you're on a tight node budget and can't afford `maxSurge`, use `maxUnavailable: 1` and make sure your app handles reduced capacity gracefully.

---

## Fix 4: preStop Hook + Graceful Shutdown

### The Problem

When Kubernetes terminates a pod, here's the sequence:

1. Pod is marked for deletion
2. Kubernetes removes the pod from the service endpoints list
3. Pod receives `SIGTERM`
4. After `terminationGracePeriodSeconds`, pod receives `SIGKILL`

The problem: steps 1 and 2 happen asynchronously. There's a race condition — the endpoint controller takes a few seconds to propagate the removal across the cluster. During that window, the load balancer might still route traffic to your terminating pod.

If your app shuts down immediately on `SIGTERM`, those in-flight requests are dropped. 502.

### The Fix

Add a `preStop` hook with a sleep:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```

And set a proper termination grace period:

```yaml
spec:
  terminationGracePeriodSeconds: 60
```

The `sleep 5` gives Kubernetes time to finish propagating the endpoint removal before the app starts shutting down. The grace period gives in-flight requests time to complete.

**Your app also needs to handle SIGTERM gracefully.** Don't just exit immediately — drain active connections first:

```python
# Python/FastAPI example
import signal, asyncio

async def shutdown(signal, loop):
    print(f"Received {signal.name}, shutting down...")
    # Stop accepting new connections
    server.should_exit = True
    # Wait for in-flight requests
    await asyncio.sleep(5)
    loop.stop()

signal.signal(signal.SIGTERM, lambda s, f: asyncio.create_task(shutdown(s, loop)))
```

---

## Putting It All Together

Here's a complete deployment snippet with all four fixes applied:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: my-app
          image: my-app:latest
          ports:
            - containerPort: 8080
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
```

Copy this, adapt the image and paths, and your rolling updates will be zero-downtime.

---

## Quick Checklist Before Every Deployment

- [ ] Startup probe configured with enough time for worst-case boot
- [ ] Readiness probe hitting a real `/ready` endpoint (not just a 200)
- [ ] `maxUnavailable: 0` in rolling update strategy
- [ ] `preStop` sleep hook added
- [ ] `terminationGracePeriodSeconds` set to at least 30s (60s is safer)
- [ ] App handles SIGTERM gracefully and drains connections

---

## Where to Go From Here

If you're managing Kubernetes in production, these tools will save you significant pain:

- **[DigitalOcean Managed Kubernetes (DOKS)](https://www.digitalocean.com/?refcode=YOUR_REF_CODE)** — The easiest managed K8s I've used. Control plane is free, and the node pricing is straightforward. Good starting point if you're moving off EKS/GKE to cut costs. *(affiliate link)*
- **[Better Stack](https://betterstack.com/?ref=goliveapp)** — Uptime monitoring + on-call alerting. Set up monitors on your `/ready` endpoint so you know about 502s before your users do. *(affiliate link)*
- **[Vultr Cloud Compute](https://www.vultr.com/?ref=YOUR_REF_CODE)** — Cheap VMs and managed K8s if you want full control at low cost. *(affiliate link)*

---

## Summary

| Problem | Fix |
|---|---|
| App not ready but receiving traffic | Add readiness probe |
| App crash-looping during slow start | Add startup probe |
| Old pods killed before new ones ready | `maxUnavailable: 0` in rolling strategy |
| In-flight requests dropped on SIGTERM | preStop sleep + graceful shutdown |

502s during deployments are a solved problem. It's just a matter of configuring what Kubernetes gives you out of the box.

Got a specific scenario that's still causing 502s? Drop it in the comments — happy to dig in.
