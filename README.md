Kubernetes on Proxmox

This repository provides documentation and best practices for running **Kubernetes clusters on Proxmox VE**.  
It covers:

- Persistent Volumes (PV) & Persistent Volume Claims (PVC)  
- Service exposure (ClusterIP vs NodePort)  
- Liveness & Readiness Probes for **zero-downtime deployments**

---

## 🚀 Environment Setup

- **Virtualization Platform:** Proxmox VE  
- **Kubernetes Nodes:** Deployed as VMs on Proxmox  
- **Networking:**  
  - Proxmox bridges (e.g., `vmbr0`, `vmbr1`)  
  - Kubernetes CNI plugins (Calico, Flannel, Cilium)

---

## 📦 Persistent Volumes (PV) & PVC

### Storage Options on Proxmox
- Local storage (bind-mounted disks or directories)  
- Ceph RBD (**recommended**)  
- NFS / iSCSI (shared storage)  
- ZFS pools  

### Example PVC (Ceph RBD)
```yaml
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
🌐 Service Types
ClusterIP (default)
Internal-only service, accessible inside the cluster.

yaml
Copy code
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
NodePort
Exposes a service on each node’s IP at a static port (30000–32767).

yaml
Copy code
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
⚠️ Best Practice: Use Ingress + LoadBalancer (via MetalLB or HAProxy) for production instead of NodePort.

🔄 Liveness & Readiness Probes
Why Probes?
Liveness Probe: Restarts unresponsive containers

Readiness Probe: Ensures pod is ready before receiving traffic

Critical for rolling updates & zero downtime

Example Deployment
yaml
Copy code
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
✅ Best Practices
Use Ceph RBD or ZFS-backed CSI drivers for PVs

Prefer ClusterIP + Ingress/MetalLB for production services

Configure readiness & liveness probes correctly

Use RollingUpdate strategy with maxUnavailable=1

Test probes locally before rollout

