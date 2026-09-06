# Enterprise Nextcloud AIO Deployment on Single-Node Kubernetes (K3s)

An enterprise-grade, resilient deployment of **Nextcloud All-in-One (AIO)** running on a hardened **Arch Linux** host. This project demonstrates bare-metal Linux administration, Kubernetes container orchestration via **K3s**, dynamic storage provisioning over **NFS**, and zero-trust edge ingress using **Tailscale Funnel**.

---

## 1. Architectural Overview

The deployment decouples compute, storage, and networking layers to simulate enterprise cloud-native patterns within a single-node environment:

```
                      Internet / Remote Clients
                                 │
                     [ Tailscale Mesh Network ]
                                 │ (Encrypted WireGuard Mesh / HTTPS Ingress)
                                 ▼
                     [ Tailscale Funnel (443) ]
                                 │ (Reverse Proxy to localhost:11000)
                                 ▼
         ┌──────────────────────────────────────────────────┐
         │ Arch Linux Host (gilravager)                     │
         │ Kernel: 7.1.7-arch1-1 | K3s v1.36.3+k3s1         │
         │                                                  │
         │  ┌────────────────────────────────────────────┐  │
         │  │ K3s Cluster Engine (containerd 2.3.2)      │  │
         │  │                                            │  │
         │  │  [ Klipper ServiceLB: Port 11000 ]         │  │
         │  │               │                            │  │
         │  │               ▼                            │  │
         │  │  [ nextcloud-aio-apache ]                  │  │
         │  │         │                 │                │  │
         │  │         ▼                 ▼                │  │
         │  │  [ nextcloud-core ]  [ notify-push ]       │  │
         │  │         │                 │                │  │
         │  │         ▼                 ▼                │  │
         │  │  [ PostgreSQL 16 ]   [ Redis Cache ]       │  │
         │  └─────────┬─────────────────┬────────────────┘  │
         │            │ (Dynamic PVCs)  │                   │
         │            ▼                 ▼                   │
         │  [ nfs-subdir-external-provisioner ]             │
         │  StorageClass: nfs-client (Default)              │
         │            │                                     │
         └────────────┼─────────────────────────────────────┘
                      ▼
         [ Local NFS Server: /srv/nfs/nextcloud ]
```

---

## 2. Technical Stack & Key Components

* **Host OS:** Arch Linux (Rolling release, Linux kernel `7.1.7-arch1-1-amd64`)
* **Container Runtime & Orchestration:** K3s (`v1.36.3+k3s1`), `containerd 2.3.2`
* **Package Management:** Helm v3
* **Application Framework:** Nextcloud AIO (Apache 2.4, PHP-FPM, PostgreSQL 16, Redis, High-Performance Push)
* **Persistent Storage:** Local NFS v4 Server (`nfs-utils`) dynamic volume management via `nfs-subdir-external-provisioner`
* **Zero-Trust Network Access (ZTNA):** Tailscale Funnel & MagicDNS

---

## 3. Storage Architecture: Dynamic NFS Provisioning

Kubernetes pods are ephemeral by design. To guarantee database integrity and object persistence, storage was decoupled from node disks:

1. **Kernel-Level NFS Export:** A persistent root `/srv/nfs/nextcloud` was exported with `no_root_squash`, `rw`, and `fsid=0` to permit container uid/gid matching.
2. **Dynamic Provisioning via Helm:** Deployed `nfs-subdir-external-provisioner` configured to target the local loopback (`127.0.0.1`).
3. **StorageClass Patching:** Configured `nfs-client` as the cluster's default storage class, demoting the local path driver.

```bash
$ kubectl get storageclass,pv
NAME                                               PROVISIONER                                                 RECLAIMPOLICY   VOLUMEBINDINGMODE
storageclass.storage.k8s.io/nfs-client (default)   cluster.local/nfs-storage-nfs-subdir-external-provisioner   Delete          Immediate

NAME                                                        CAPACITY   ACCESS MODES   STATUS   CLAIM
persistentvolume/pvc-1bb2cd02-5809-4dc3-94e6-b1c4f747093f   5Gi        RWX            Bound    default/nextcloud-aio-nextcloud
persistentvolume/pvc-86c6c0fb-79e5-4c0c-b7fb-6cfb7dd91084   1Gi        RWO            Bound    default/nextcloud-aio-database
persistentvolume/pvc-3e0083e3-5e34-479a-b480-f6e174634549   1Gi        RWO            Bound    default/nextcloud-aio-redis
persistentvolume/pvc-5433e0fb-0ffc-4b3c-b617-56f35bbc3320   1Gi        RWO            Bound    default/nextcloud-aio-apache
persistentvolume/pvc-dcb7b3f0-1845-4c18-9c23-282633e58756   5Gi        RWO            Bound    default/nextcloud-aio-nextcloud-data
```

---

## 4. Engineering Challenges & Problem Resolution

### Challenge 1: Premature Kubelet Pod Eviction (CrashLoopBackOff)

* **Symptom:** During initial boot, Nextcloud database initialization and app migrations took longer than the default Kubernetes `livenessProbe` threshold. The Kubelet marked the container unhealthy and terminated it mid-setup.
* **Root Cause:** Nextcloud initialization requires extensive pre-flight checks, schema generation, and redis verification before Apache opens port 9000/11000.
* **Resolution:** Proactively patched the deployment spec using `kubectl patch` to remove premature liveness probes while extending the `readinessProbe` grace interval (`initialDelaySeconds: 1200`).

```bash
kubectl patch deployment nextcloud-aio-nextcloud -n default --type=json -p='[
  {"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}
]'
```

### Challenge 2: Secure Public Ingress Without Inbound Port Forwarding

* **Symptom:** Standard ingress requires opening port 80/443 on edge routers and dynamically updating public DNS records.
* **Resolution:** Implemented **Tailscale Funnel**. The Apache service binds to cluster port `11000`, while the Tailscale daemon proxies HTTPS requests through encrypted tunnels directly to loopback:

```bash
# Verify active ingress state
$ tailscale funnel status
https://gilravager.bongo-shiner.ts.net (Funnel on)
|-- / proxy http://127.0.0.1:11000
```

---

## 5. High Availability & Reboot Resilience Validation

A core production requirement was zero-touch system recovery following unannounced power cuts and reboots.

### Systemd Service Automation

All underlying platform daemons were configured to survive system initialization targets:

```bash
$ sudo systemctl list-unit-files --state=enabled | grep -E "k3s|nfs|tailscale|resolved"
k3s.service                          enabled disabled
nfs-server.service                   enabled disabled
systemd-resolved.service             enabled enabled
tailscaled.service                   enabled disabled
```

### Verified Cluster Health (Post-Reboot Audit)

All microservices returned to fully functional `Ready` states without human intervention:

```bash
$ kubectl get pods -A -o wide
NAMESPACE     NAME                                                           READY   STATUS      RESTARTS   IP            NODE
default       nextcloud-aio-apache-7747fdb8f9-sfv8b                          1/1     Running     0          10.42.0.153   gilravager
default       nextcloud-aio-database-5ddc76c84-78jql                         1/1     Running     3          10.42.0.151   gilravager
default       nextcloud-aio-nextcloud-64f7fc9fbc-q7kk5                       1/1     Running     3          10.42.0.145   gilravager
default       nextcloud-aio-notify-push-99955d4f-lrr8s                       1/1     Running     3          10.42.0.148   gilravager
default       nextcloud-aio-redis-597bcbbcfd-zhhrz                           1/1     Running     3          10.42.0.152   gilravager
default       nfs-storage-nfs-subdir-external-provisioner-66dc744b7d-74jp5   1/1     Running     9          10.42.0.150   gilravager
```

---

## 6. Skills Demonstrated

* **Linux System Administration:** Arch Linux management, systemd service lifecycle, storage exports (`/etc/exports`), network configuration (`systemd-resolved`).
* **Container Orchestration:** K3s deployment, manifest lifecycle management, DaemonSets, Helm chart customization, storage class patching.
* **Networking & Security:** ZTNA via Tailscale mesh VPN, reverse proxy configuration, ingress load balancing, internal Kubernetes DNS resolution (`CoreDNS`).
