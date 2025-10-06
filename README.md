# Kubernetes-on-Proxmox-Deployment-Best-Practices
•  Kubernetes on Proxmox: PV/PVC storage, ClusterIP/NodePort, liveness &amp; readiness probes for zero-downtime.

Kubernetes on Proxmox: Deployment & Best Practices
1. Introduction

This document provides guidance on deploying and managing Kubernetes clusters on Proxmox VE. It covers persistent storage (PV/PVC), service exposure (ClusterIP vs NodePort), and implementing liveness/readiness probes to ensure high availability and zero-downtime application updates.

2. Environment Setup on Proxmox

Proxmox VE is used as the virtualization platform.

Kubernetes nodes (masters & workers) are deployed as VMs on Proxmox.

Networking can be handled via:

Proxmox bridges (vmbr0, vmbr1, etc.)

CNI plugin inside Kubernetes (e.g., Calico, Flannel, Cilium).

3. Persistent Volumes (PV) & Persistent Volume Claims (PVC)
3.1 Storage Options on Proxmox

Local storage: Bind-mounted disks or directories.

Proxmox storage backends:

Ceph RBD (recommended for production, distributed & replicated)

NFS/iSCSI (shared storage)

ZFS pools (fast, resilient)

3.2 PV/PVC Workflow

Define a PersistentVolume (PV) – Represents physical storage.

Create a PersistentVolumeClaim (PVC) – Request for storage by a pod.

Kubelet attaches the storage automatically if PV matches PVC requirements.

Example PVC with Ceph RBD:

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-rbd
  resources:
    requests:
      storage: 5Gi

4. Service Types: ClusterIP vs NodePort
4.1 ClusterIP (default)

Internal-only service.

Pods can communicate using the service DNS name.

Use when services should not be directly exposed outside the cluster.

apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 8080
  type: ClusterIP

4.2 NodePort

Exposes service on each node’s IP at a static port (30000–32767).

Useful for development or when external load balancers are not available.

apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      nodePort: 30080
  type: NodePort


Best Practice: Use Ingress + LoadBalancer (via MetalLB or HAProxy on Proxmox) for production instead of relying on NodePort.

5. Liveness & Readiness Probes for Zero-Downtime
5.1 Why Probes?

Liveness Probe: Ensures the container is healthy; restarts it if unresponsive.

Readiness Probe: Ensures the pod is ready before it starts receiving traffic.

Essential for rolling updates and zero-downtime deployments.

5.2 Example Deployment with Probes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app-container
          image: myregistry/my-app:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 20

6. Best Practices

Storage: Use Ceph RBD or ZFS-backed CSI drivers for dynamic PV provisioning in Proxmox.

Services: Prefer ClusterIP + Ingress/MetalLB for production-grade service exposure.

Zero Downtime:

Configure probes properly.

Use RollingUpdate strategy with maxUnavailable=1.

Always test probes locally before production rollout.

7. Conclusion

Running Kubernetes on Proxmox provides flexibility for homelab and production environments. By configuring PV/PVC for persistent storage, properly exposing services, and using probes, you can ensure scalability, resiliency, and zero downtime for your workloads.



