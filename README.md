# Enterprise Nextcloud AIO Deployment on Single-Node Kubernetes (K3s)

An enterprise-grade, resilient deployment of **Nextcloud All-in-One (AIO)** running on a hardened **Arch Linux** host. This project demonstrates bare-metal Linux administration, Kubernetes container orchestration via **K3s**, dynamic storage provisioning over **NFS**, and zero-trust edge ingress using **Tailscale Funnel**.

---

## 1. Architectural Overview

The deployment decouples compute, storage, and networking layers to simulate enterprise cloud-native patterns within a single-node environment:

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

---

## 2. Technical Stack & Key Components

* **Host OS:** Arch Linux (Rolling release, Linux kernel `7.1.7-arch1-1-amd64`)[cite: 8]
* **Container Runtime & Orchestration:** K3s (`v1.36.3+k3s1`), `containerd 2.3.2`[cite: 8]
* **Package Management:** Helm v3
* **Application Framework:** Nextcloud AIO (Apache 2.4, PHP-FPM, PostgreSQL 16, Redis, High-Performance Push)
* **Persistent Storage:** Local NFS v4 Server (`nfs-utils`) dynamic volume management via `nfs-subdir-external-provisioner`[cite: 4, 6]
* **Zero-Trust Network Access (ZTNA):** Tailscale Funnel & MagicDNS

---

## 3. Storage Architecture: Dynamic NFS Provisioning

Kubernetes pods are ephemeral by design. To guarantee database integrity and object persistence, storage was decoupled from node disks:

1. **Kernel-Level NFS Export:** A persistent root `/srv/nfs/nextcloud` was exported with `no_root_squash`, `rw`, and `fsid=0` to permit container uid/gid matching[cite: 11].
2. **Dynamic Provisioning via Helm:** Deployed `nfs-subdir-external-provisioner` configured to target the local loopback (`127.0.0.1`)[cite: 4].
3. **StorageClass Patching:** Configured `nfs-client` as the cluster's default storage class, demoting the local path driver[cite: 6].

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

