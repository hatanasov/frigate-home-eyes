# Home Surveillance Stack on Kubernetes

A self-hosted, Kubernetes-native home surveillance system built from three open-source components:

- **Frigate NVR** — AI-powered network video recorder with object detection
- **Eclipse Mosquitto** — lightweight MQTT message broker
- **Frigate-Notify** — alert forwarder that reads MQTT events and sends notifications to Discord

## Architecture Overview

```
IP Cameras (RTSP)
      │
      ▼
┌─────────────┐     MQTT events      ┌───────────────────┐     Discord Webhook
│  Frigate NVR│ ──────────────────▶  │ Eclipse Mosquitto │ ◀──────────────────
│  (detection)│                      │  (message broker) │
└─────────────┘                      └───────────────────┘
                                              │
                                              ▼
                                    ┌──────────────────────┐
                                    │   Frigate-Notify     │
                                    │  (alert dispatcher)  │
                                    └──────────────────────┘
                                              │
                                              ▼
                                         Discord Channel
```

When a camera detects a person entering a defined zone, Frigate publishes an MQTT event to Mosquitto. Frigate-Notify subscribes to those events and forwards them — with a snapshot — to a Discord channel via webhook.

---

## File Reference

### `frigate-claim2-persistentvolumeclaim.yaml`
Defines storage for Frigate:

- **`frigate-storage-volume`** — a `PersistentVolume` backed by a host path (`/media/frigate-data`) for storing recordings. Sized at 276 Gi; adjust to match your disk.
- **`frigate-claim-storage`** — PVC that binds to the above PV for recording storage.
- **`frigate-claim-config`** — a small (50 Mi) PVC for the Frigate config file, populated by an init container at startup.

---

### `frigate-cm0-configmap.yaml`
The main Frigate configuration, mounted into the pod via an init container. Key sections:

- **MQTT** — connects to the in-cluster Mosquitto service (`mqtt:1883`) using credentials that must match those in `frigate-notify-secret.yaml` and `mqtt-cm1-configmap.yaml`.
- **Detector** — uses OpenVINO on CPU (`ssdlite_mobilenet_v2`) for object detection.
- **Objects** — tracks `person` only.
- **Record** — recordings retained for 0 days by default; alert clips (active objects) kept for 30 days.
- **Review / Alerts** — alerts fire when a person enters `secure_1` or `secure_2` zones.
- **Cameras** — two RTSP camera inputs (`cam_1_name`, `cam_2_name`). Fill in real IPs, usernames, and passwords before deploying.

> ⚠️ **Security — see section below.**

---

### `frigate-deployment.yaml`
Deploys the Frigate NVR container (`ghcr.io/blakeblackshear/frigate:0.16.0`).

- **Init container** (`busybox`) copies the ConfigMap config into the config PVC before Frigate starts.
- **Volumes** — tmpfs cache (1 Gi RAM), config PVC, storage PVC, `/dev/dri` for hardware-accelerated decoding, and a shared memory volume (512 Mi).
- **Resources** — limited to 2 CPU cores and 4 Gi RAM.
- **`privileged: true`** is required for hardware device access (`/dev/dri`).

---

### `frigate-notify-secret.yaml`
A Kubernetes `Secret` holding credentials consumed by the `frigate-notify` deployment as environment variables:

| Key | Purpose |
|-----|---------|
| `FN_FRIGATE__MQTT__USERNAME` | MQTT username (must match Mosquitto and Frigate config) |
| `FN_FRIGATE__MQTT__PASSWORD` | MQTT password |
| `FN_FRIGATE__USERNAME` | Frigate web UI username |
| `FN_FRIGATE__PASSWORD` | Frigate web UI password |

Values must be **base64-encoded**. Example:
```bash
echo -n 'mysecretpassword' | base64
```

> ⚠️ **Do not commit real values to version control — see security section below.**

---

### `frigate-notify.yaml`
Deploys `frigate-notify` (`ghcr.io/0x2142/frigate-notify:latest`), which bridges MQTT alerts to Discord.

- Credentials are injected from the `frigate-notify` Secret (not hardcoded).
- Connects to Frigate at `http://frigate.default.svc.cluster.local:5000`.
- Connects to Mosquitto at `mqtt.default.svc.cluster.local:1883`.
- Sends alerts to Discord via a webhook URL configured in `FN_ALERTS__DISCORD__WEBHOOK`.

> ⚠️ The Discord webhook URL is currently a plaintext environment variable — see security section.

---

### `frigate-service.yaml`
Exposes Frigate via `NodePort`:

| Port | Purpose |
|------|---------|
| 5000 → 30005 | Frigate web UI / API |
| 1935 → 31935 | RTMP stream |
| 8971 → 31798 | Frigate HTTPS UI |
| 8554 → 30554 | RTSP restream |

---

### `ingeress.yaml`
An Ingress resource (Traefik) that routes `frigate.home` → Frigate service port 5000 over plain HTTP (`web` entrypoint).

> ⚠️ Currently HTTP only — see security section.

---

### `mqtt-cm0-configmap.yaml`
Mosquitto broker configuration:

- Anonymous access **disabled** (`allow_anonymous false`) ✅
- Password file at `/mosquitto/config/password.txt`
- Listens on port 1883 (plain MQTT, no TLS)

---

### `mqtt-cm1-configmap.yaml`
Contains the Mosquitto password file. Passwords must be **pre-hashed** using the `mosquitto_passwd` tool:

```bash
mosquitto_passwd -c /tmp/password.txt <username>
# then copy the hashed line into the ConfigMap
```

> ⚠️ This file is stored as a ConfigMap (not a Secret) — see security section.

---

### `mqtt-deployment.yaml`
Deploys Eclipse Mosquitto (`eclipse-mosquitto`).

- Mounts `mqtt-cm0` (broker config) and `mqtt-cm1` (password file) using `subPath` mounts so only the specific files are overwritten inside the container.
- Resources limited to 128m CPU and 64 Mi RAM.

> ⚠️ No image tag pinned — see security section.

---

### `mqtt-service.yaml`
Exposes Mosquitto via `NodePort` on port 31883. This makes the MQTT broker reachable from outside the cluster.

> ⚠️ Consider whether external MQTT exposure is necessary — see security section.

## Deployment Order

```bash
# 1. Storage
kubectl apply -f frigate-claim2-persistentvolumeclaim.yaml

# 2. Secrets (fill in real base64 values first)
kubectl apply -f frigate-notify-secret.yaml

# 3. ConfigMaps
kubectl apply -f frigate-cm0-configmap.yaml
kubectl apply -f mqtt-cm0-configmap.yaml
kubectl apply -f mqtt-cm1-configmap.yaml

# 4. Deployments
kubectl apply -f mqtt-deployment.yaml
kubectl apply -f frigate-deployment.yaml
kubectl apply -f frigate-notify.yaml

# 5. Services & Ingress
kubectl apply -f mqtt-service.yaml
kubectl apply -f frigate-service.yaml
kubectl apply -f ingeress.yaml
```

---

## Prerequisites

- Kubernetes cluster (tested on k3s)
- Traefik ingress controller
- Node with `/dev/dri` (Intel GPU / iGPU) for hardware-accelerated decoding
- Local storage provisioner (`local-path`)
- IP cameras with RTSP streams
- Discord server with an incoming webhook configured

## References

- https://docs.frigate.video/
- https://frigate-notify.0x2142.com/latest/   
- https://mosquitto.org/  
