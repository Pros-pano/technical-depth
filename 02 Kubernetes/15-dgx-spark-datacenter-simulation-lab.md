# 15. DGX Spark Data Center Simulation Lab — Step-by-Step Implementation

This is the central hands-on implementation guide for partitioning your bare-metal **DGX Spark** host into a dual-tenant, data-center-like environment.

Following your specifications:
- **No VMware hypervisor**: Direct bare-metal Linux OS execution with zero virtualization tax.
- **Two isolated environments**: Simulating two independent cluster/tenant zones (`k3s-alpha` and `k3s-beta`).
- **Strict 5% allocation**: Exactly **5% compute (CPU/RAM)** and **5% storage** allocated to each environment (leaving 90% reserved for host OS stability, background monitoring, and other infrastructure).
- **GPU acceleration**: Both environments access the NVIDIA GPU simultaneously via **Time-Slicing**.

---

## 📑 Table of Contents
1. [Architecture Overview & Resource Budgeting](#1-architecture-overview--resource-budgeting)
2. [Step 1: Host OS Preparation & NVIDIA Container Toolkit](#step-1-host-os-preparation--nvidia-container-toolkit)
3. [Step 2: Install K3s with GPU Runtime Integration](#step-2-install-k3s-with-gpu-runtime-integration)
4. [Step 3: Deploy the NVIDIA Device Plugin with Time-Slicing](#step-3-deploy-the-nvidia-device-plugin-with-time-slicing)
5. [Step 4: Provision Dual Tenant Namespaces with 5% Quotas](#step-4-provision-dual-tenant-namespaces-with-5-quotas)
6. [Step 5: Enforce Storage Isolation (5% Local Path Quotas)](#step-5-enforce-storage-isolation-5-local-path-quotas)
7. [Step 6: Deploy & Benchmark AI Workloads](#step-6-deploy--benchmark-ai-workloads)
8. [Step 7: Advanced Architecture — Dual Independent K3s Systemd Slices](#step-7-advanced-architecture--dual-independent-k3s-systemd-slices)
9. [Verification & Health Checklist](#verification--health-checklist)

---

## 1. Architecture Overview & Resource Budgeting

```text
+---------------------------------------------------------------------------------------------------+
|                                 DGX Spark Host (Linux OS Bare-Metal)                              |
|                          Hardware: Grace CPU + Blackwell GPU (Unified Memory)                     |
|                                                                                                   |
|  +---------------------------------------------------------------------------------------------+  |
|  | K3s Kubernetes Engine (configured with default-runtime = nvidia-container-runtime)           |  |
|  +---------------------------------------------------------------------------------------------+  |
|          |                                                                             |          |
|          v                                                                             v          |
|  +---------------------------------------------+               +-------------------------------+  |
|  | Namespace: `k3s-alpha` (Tenant 1)           |               | Namespace: `k3s-beta` (Tenant 2) |
|  | ├── ResourceQuota: 5% CPU, 5% RAM, 1 GPU    |               | ├── ResourceQuota: 5% CPU, RAM|  |
|  | ├── Storage: 5% Disk Quota (PVCs)           |               | ├── Storage: 5% Disk Quota    |  |
|  | └── Workload: PyTorch Training / Fine-tuning|               | └── Workload: Triton Inference|  |
|  +---------------------------------------------+               +-------------------------------+  |
|          \                                                                             /          |
|           +-------------------------------------+-------------------------------------+           |
|                                                 |                                                 |
|                                                 v                                                 |
|                             Host System Reserve: 90% CPU, RAM, NVMe                               |
|                         (Protects `nvidia-smi`, host cron, ssh, and OS)                           |
+---------------------------------------------------------------------------------------------------+
```

### Resource Sizing Table (Dynamic Math)
Run `lscpu`, `free -m`, and `df -h /` on your DGX Spark to confirm exact values. Below is an example based on a typical 64-core Grace CPU with 128GB unified memory and 1TB NVMe:

| Metric | Total Host Available | 5% Allocation per Tenant | Enforcement Mechanism |
| :--- | :--- | :--- | :--- |
| **CPU** | 64 Cores (`64000m`) | **3.2 Cores (`3200m`)** | Kubernetes `ResourceQuota` (`limits.cpu: 3200m`) |
| **Memory** | 128 GiB Unified RAM | **6.4 GiB (`6553Mi`)** | Kubernetes `ResourceQuota` (`limits.memory: 6400Mi`) |
| **NVMe Disk** | 1,000 GiB (1 TB) | **50 GiB** | StorageClass PVC Quota (`requests.storage: 50Gi`) |
| **GPU Compute** | 1 Physical Blackwell GPU | **1 Time-Slice (1/10th)** | NVIDIA Device Plugin Time-Slicing ConfigMap |

---

## Step 1: Host OS Preparation & NVIDIA Container Toolkit

Log into your DGX Spark machine (`dgx-spark-1` via SSH) as `dgxadmin` with `sudo` privileges.

### 1.1 Verify Driver and Kernel Status
```bash
# Verify GPU detection
nvidia-smi

# Verify kernel modules are loaded
lsmod | grep nvidia
```

### 1.2 Install NVIDIA Container Toolkit
If not already installed, add the official repository and install the toolkit:
```bash
# 1. Setup repository
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 2. Install packages
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# 3. Configure Container Device Interface (CDI)
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
nvidia-ctk cdi list
```

---

## Step 2: Install K3s with GPU Runtime Integration

K3s is a lightweight, fully compliant Kubernetes distribution designed for bare-metal edge and single-node clusters. It runs using less than 512MB RAM for the control plane.

### 2.1 Pre-configure containerd to use NVIDIA as Default Runtime
K3s manages its own embedded `containerd`. We create a configuration template so K3s automatically registers the `nvidia` runtime:

```bash
sudo mkdir -p /var/lib/rancher/k3s/agent/etc/containerd/

sudo tee /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl << 'EOF'
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  BinaryName = "/usr/bin/nvidia-container-runtime"

[plugins."io.containerd.grpc.v1.cri".containerd]
  default_runtime_name = "runc"
EOF
```

### 2.2 Install K3s
Install K3s with Traefik disabled (saves memory and avoids port conflicts) and local storage enabled:
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --write-kubeconfig-mode 644" sh -
```

### 2.3 Verify K3s Cluster Health
```bash
# Check node status
kubectl get nodes -o wide

# Check core pods
kubectl get pods -A
```

---

## Step 3: Deploy the NVIDIA Device Plugin with Time-Slicing

By default, Kubernetes assigns 1 physical GPU exclusively to 1 Pod. To allow both `k3s-alpha` and `k3s-beta` to share the GPU, we configure the **NVIDIA Kubernetes Device Plugin with Time-Slicing**.

### 3.1 Create Time-Slicing ConfigMap
Create a file named `time-slicing-config.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nvidia-device-plugin-config
  namespace: kube-system
data:
  any: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        resources:
          - name: nvidia.com/gpu
            replicas: 10
            renameByDefault: false
            failRequestsGreaterThanOne: false
```
*Explanation: This divides the single physical GPU into **10 virtual time-slices**. Each tenant will request 1 slice (10% of scheduling time).*

Apply the ConfigMap:
```bash
kubectl apply -f time-slicing-config.yaml
```

### 3.2 Deploy the NVIDIA Device Plugin via DaemonSet
Create a file named `nvidia-device-plugin.yaml`:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - image: nvcr.io/nvidia/k8s-device-plugin:v0.14.5
          name: nvidia-device-plugin-ctr
          env:
            - name: CONFIG_FILE
              value: /etc/config/any
          volumeMounts:
            - name: device-plugin
              mountPath: /var/lib/kubelet/device-plugins
            - name: config
              mountPath: /etc/config
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
        - name: config
          configMap:
            name: nvidia-device-plugin-config
```

Apply the DaemonSet:
```bash
kubectl apply -f nvidia-device-plugin.yaml
```

### 3.3 Verify Advertised GPU Slices
Wait 10 seconds, then inspect your node's allocatable capacity:
```bash
kubectl get node -o jsonpath='{.items[0].status.allocatable.nvidia\.com/gpu}'
```
*Expected output: `10` (The single GPU is now registered as 10 schedulable slices).*

---

## Step 4: Provision Dual Tenant Namespaces with 5% Quotas

Now we create the two tenant environments: **`k3s-alpha`** (Tenant 1) and **`k3s-beta`** (Tenant 2).

### 4.1 Create Namespaces
```bash
kubectl create namespace k3s-alpha
kubectl create namespace k3s-beta
```

### 4.2 Define 5% ResourceQuota for Tenant Alpha
Create `quota-alpha.yaml`:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota-5pct
  namespace: k3s-alpha
spec:
  hard:
    # 5% of 64 cores = 3200m (3.2 cores)
    requests.cpu: "1600m"
    limits.cpu: "3200m"
    
    # 5% of 128GB RAM = 6400Mi (6.4 GiB)
    requests.memory: "3200Mi"
    limits.memory: "6400Mi"
    
    # 1 GPU Time-Slice
    requests.nvidia.com/gpu: "1"
    limits.nvidia.com/gpu: "1"
    
    # 5% of 1TB Storage = 50Gi
    requests.storage: "50Gi"
    persistentvolumeclaims: "4"
    
    # Maximum Pods
    pods: "5"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: limits-alpha
  namespace: k3s-alpha
spec:
  limits:
    - default:
        cpu: "1000m"
        memory: "2048Mi"
        nvidia.com/gpu: "1"
      defaultRequest:
        cpu: "500m"
        memory: "1024Mi"
        nvidia.com/gpu: "1"
      max:
        cpu: "3200m"
        memory: "6400Mi"
      type: Container
```

Apply for `k3s-alpha`:
```bash
kubectl apply -f quota-alpha.yaml
```

### 4.3 Define 5% ResourceQuota for Tenant Beta
Create `quota-beta.yaml`:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota-5pct
  namespace: k3s-beta
spec:
  hard:
    requests.cpu: "1600m"
    limits.cpu: "3200m"
    requests.memory: "3200Mi"
    limits.memory: "6400Mi"
    requests.nvidia.com/gpu: "1"
    limits.nvidia.com/gpu: "1"
    requests.storage: "50Gi"
    persistentvolumeclaims: "4"
    pods: "5"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: limits-beta
  namespace: k3s-beta
spec:
  limits:
    - default:
        cpu: "1000m"
        memory: "2048Mi"
        nvidia.com/gpu: "1"
      defaultRequest:
        cpu: "500m"
        memory: "1024Mi"
        nvidia.com/gpu: "1"
      max:
        cpu: "3200m"
        memory: "6400Mi"
      type: Container
```

Apply for `k3s-beta`:
```bash
kubectl apply -f quota-beta.yaml
```

---

## Step 5: Enforce Storage Isolation (5% Local Path Quotas)

K3s comes with Rancher's **Local Path Provisioner** pre-installed. It dynamically allocates storage from the host's `/var/lib/rancher/k3s/storage` directory.

### 5.1 Create Persistent Volume Claims
Each tenant creates a PVC bounded by their 50Gi (5%) quota.

Create `storage-alpha.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-volume-alpha
  namespace: k3s-alpha
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 20Gi  # Consumes 20Gi out of the 50Gi quota
```

Create `storage-beta.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-volume-beta
  namespace: k3s-beta
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 20Gi  # Consumes 20Gi out of the 50Gi quota
```

Apply both:
```bash
kubectl apply -f storage-alpha.yaml
kubectl apply -f storage-beta.yaml
```

Verify storage bindings:
```bash
kubectl get pvc -A
```

---

## Step 6: Deploy & Benchmark AI Workloads

Now we deploy active AI workloads into both tenants to prove:
1. Both workloads access the NVIDIA GPU simultaneously.
2. Both workloads run within their 5% compute envelope.

### 6.1 Tenant Alpha: PyTorch GPU Tensor Benchmark
Create `workload-alpha.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pytorch-benchmark
  namespace: k3s-alpha
spec:
  restartPolicy: Never
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: data-volume-alpha
  containers:
    - name: pytorch
      image: nvcr.io/nvidia/pytorch:24.01-py3
      volumeMounts:
        - name: storage
          mountPath: /data
      resources:
        requests:
          cpu: "1000m"
          memory: "2048Mi"
          nvidia.com/gpu: "1"
        limits:
          cpu: "2000m"
          memory: "4096Mi"
          nvidia.com/gpu: "1"
      command: ["python3", "-c"]
      args:
        - |
          import torch, time
          print("=== Tenant Alpha: PyTorch GPU Validation ===")
          print("Device Name:", torch.cuda.get_device_name(0))
          print("Allocating 10,000 x 10,000 Float32 Matrix on GPU...")
          x = torch.randn(10000, 10000, device='cuda')
          start = time.time()
          for i in range(100):
              y = torch.matmul(x, x)
          torch.cuda.synchronize()
          elapsed = time.time() - start
          print(f"100 MatMuls Completed in: {elapsed:.2f} seconds")
          with open('/data/benchmark_results.txt', 'w') as f:
              f.write(f"Alpha finished 100 MatMuls in {elapsed:.2f}s on {torch.cuda.get_device_name(0)}\n")
          print("Results written to persistent volume /data.")
          time.sleep(300)
```

Apply Tenant Alpha workload:
```bash
kubectl apply -f workload-alpha.yaml
```

### 6.2 Tenant Beta: NVIDIA CUDA Inference / Stress Workload
Create `workload-beta.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-stress
  namespace: k3s-beta
spec:
  restartPolicy: Never
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: data-volume-beta
  containers:
    - name: cuda-worker
      image: nvcr.io/nvidia/k8s/cuda-sample:vectorAdd-cuda11.7.1-ubuntu20.04
      volumeMounts:
        - name: storage
          mountPath: /data
      resources:
        requests:
          cpu: "1000m"
          memory: "2048Mi"
          nvidia.com/gpu: "1"
        limits:
          cpu: "2000m"
          memory: "4096Mi"
          nvidia.com/gpu: "1"
```

Apply Tenant Beta workload:
```bash
kubectl apply -f workload-beta.yaml
```

---

## Step 7: Advanced Architecture — Dual Independent K3s Systemd Slices

If your goal is to simulate **two completely separate physical clusters** (e.g. Cluster 1 on `localhost:6443` and Cluster 2 on `localhost:6444`) rather than namespaces in a single cluster, you can run two separate K3s systemd services restricted by Linux cgroups directly:

```text
+-----------------------------------------------------------------------------------+
|                        DGX Spark Host OS Systemd CGroup v2                        |
|                                                                                   |
|  [k3s-alpha.service]                      [k3s-beta.service]                      |
|  - Listen Port: 6443                      - Listen Port: 6444                     |
|  - CPUQuota=320% (3.2 cores = 5%)         - CPUQuota=320% (3.2 cores = 5%)        |
|  - MemoryMax=6.4G (5% RAM)                - MemoryMax=6.4G (5% RAM)               |
|  - DataDir: /var/lib/k3s-alpha (50GB)     - DataDir: /var/lib/k3s-beta (50GB)     |
+-----------------------------------------------------------------------------------+
```

### Systemd Drop-in Overrides for 5% Host CGroup Limits:

1. **Create systemd override for Cluster Alpha (`/etc/systemd/system/k3s-alpha.service.d/override.conf`)**:
   ```ini
   [Service]
   # Enforce 5% Host CPU and Memory Ceilings via Linux Kernel cgroups v2
   CPUAccounting=yes
   CPUQuota=320%
   MemoryAccounting=yes
   MemoryMax=6.4G
   ```

2. **Create systemd override for Cluster Beta (`/etc/systemd/system/k3s-beta.service.d/override.conf`)**:
   ```ini
   [Service]
   CPUAccounting=yes
   CPUQuota=320%
   MemoryAccounting=yes
   MemoryMax=6.4G
   ```

3. **Reload systemd**:
   ```bash
   sudo systemctl daemon-reload
   ```

*This guarantees that even if a runaway process inside K3s spawns 1,000 threads, the Linux kernel scheduler strictly caps its CPU execution at 3.2 cores and kills it if memory exceeds 6.4 GB!*

---

## Verification & Health Checklist

Run these commands to prove your multi-tenant DGX Spark simulator is operating correctly:

### 1. Verify Pod Statuses Across Both Tenants
```bash
kubectl get pods -A -o wide
```
*Expected: Both `pytorch-benchmark` in `k3s-alpha` and `cuda-stress` in `k3s-beta` show status `Running` or `Completed`.*

### 2. Inspect Quota Consumption
```bash
# Check Tenant Alpha Quota usage:
kubectl get resourcequota compute-quota-5pct -n k3s-alpha

# Check Tenant Beta Quota usage:
kubectl get resourcequota compute-quota-5pct -n k3s-beta
```
*Output displays exact consumed vs. hard limit values for CPU, Memory, and GPUs.*

### 3. Check Real-Time GPU Utilization During Matrix Multiplication
```bash
nvidia-smi
```
*You will see the Python process running with active GPU compute memory.*

### 4. Inspect PyTorch Tensor Results
```bash
kubectl logs pytorch-benchmark -n k3s-alpha
```
*Expected output:*
```text
=== Tenant Alpha: PyTorch GPU Validation ===
Device Name: NVIDIA Blackwell / GB10
Allocating 10,000 x 10,000 Float32 Matrix on GPU...
100 MatMuls Completed in: 1.42 seconds
Results written to persistent volume /data.
```

### 5. Inspect Persistent Storage File on Host
```bash
sudo ls -lh /var/lib/rancher/k3s/storage/pvc*
sudo cat /var/lib/rancher/k3s/storage/pvc*/benchmark_results.txt
```

---

Proceed to [**16-nvidia-gpu-operator-and-network-operator.md**](16-nvidia-gpu-operator-and-network-operator.md) to explore enterprise cluster automation via Helm, the GPU Operator, and the Network Operator.
