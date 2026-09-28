# 20. Hands-On Practice Exercises & Mastery Workbook — 20 Production Challenges

This workbook contains **20 comprehensive, hands-on practice exercises** designed to build muscle memory and production expertise across all layers of the Kubernetes and NVIDIA AI infrastructure stack.

---

## 📑 Table of Contents
1. [Exercise 01: Linux Namespace & cgroup v2 Inspection on Bare-Metal](#exercise-01-linux-namespace--cgroup-v2-inspection-on-bare-metal)
2. [Exercise 02: Raw etcd Key-Value Inspection via etcdctl](#exercise-02-raw-etcd-key-value-inspection-via-etcdctl)
3. [Exercise 03: API Priority & Fairness Custom Concurrency Rules](#exercise-03-api-priority--fairness-custom-concurrency-rules)
4. [Exercise 04: Production RBAC Role for an AI Engineering Team](#exercise-04-production-rbac-role-for-an-ai-engineering-team)
5. [Exercise 05: Validating Admission Webhook Policy Enforcement](#exercise-05-validating-admission-webhook-policy-enforcement)
6. [Exercise 06: Tracing Controller Manager Reconciliation Cascades](#exercise-06-tracing-controller-manager-reconciliation-cascades)
7. [Exercise 07: Advanced GPU Scheduling: Taints & Tolerations](#exercise-07-advanced-gpu-scheduling-taints--tolerations)
8. [Exercise 08: Distributed Training Gang Scheduling with Kueue](#exercise-08-distributed-training-gang-scheduling-with-kueue)
9. [Exercise 09: Tracing Virtual Ethernet (veth) Packet Flows on Host](#exercise-09-tracing-virtual-ethernet-veth-packet-flows-on-host)
10. [Exercise 10: iptables NAT Inspection for a ClusterIP Service](#exercise-10-iptables-nat-inspection-for-a-clusterip-service)
11. [Exercise 11: CoreDNS Query Tracing & Fixing the ndots:5 AI Latency Bug](#exercise-11-coredns-query-tracing--fixing-the-ndots5-ai-latency-bug)
12. [Exercise 12: Ingress Layer 7 Path Routing with Real-Time Streaming](#exercise-12-ingress-layer-7-path-routing-with-real-time-streaming)
13. [Exercise 13: StatefulSet Vector Database with Deterministic Storage](#exercise-13-statefulset-vector-database-with-deterministic-storage)
14. [Exercise 14: Indexed Job for Sharded AI Dataset Preprocessing](#exercise-14-indexed-job-for-sharded-ai-dataset-preprocessing)
15. [Exercise 15: High-Speed Local NVMe Storage with WaitForFirstConsumer](#exercise-15-high-speed-local-nvme-storage-with-waitforfirstconsumer)
16. [Exercise 16: Enforcing Hard 5% Compute and Storage Quotas on a Tenant](#exercise-16-enforcing-hard-5-compute-and-storage-quotas-on-a-tenant)
17. [Exercise 17: Generating and Validating Host CDI Device Profiles](#exercise-17-generating-and-validating-host-cdi-device-profiles)
18. [Exercise 18: Configuring NVIDIA Device Plugin with 10 GPU Time-Slices](#exercise-18-configuring-nvidia-device-plugin-with-10-gpu-time-slices)
19. [Exercise 19: Running a Multi-Tenant PyTorch GPU Matrix Benchmark](#exercise-19-running-a-multi-tenant-pytorch-gpu-matrix-benchmark)
20. [Exercise 20: Full Disaster Recovery: Restoring etcd from Snapshot](#exercise-20-full-disaster-recovery-restoring-etcd-from-snapshot)

---

### Exercise 01: Linux Namespace & cgroup v2 Inspection on Bare-Metal
- **Objective**: Prove that containers are host processes bounded by Linux namespaces and cgroups.
- **Commands**:
  ```bash
  kubectl run inspect-box --image=alpine -- sleep 3600
  CONTAINER_ID=$(sudo crictl ps --name inspect-box -q)
  PID=$(sudo crictl inspect $CONTAINER_ID | jq '.info.pid')
  echo "Host PID is: $PID"
  sudo ls -l /proc/$PID/ns/
  cat /sys/fs/cgroup$(cat /proc/$PID/cgroup | cut -d: -f3)/memory.max
  ```
- **Validation**: Verify that the process possesses independent `ipc`, `net`, `mnt`, and `pid` namespaces and that `memory.max` reflects host limits. Clean up: `kubectl delete pod inspect-box`.

---

### Exercise 02: Raw etcd Key-Value Inspection via etcdctl
- **Objective**: Interrogate the raw persistent key-value store of Kubernetes.
- **Commands**:
  ```bash
  ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    get /registry/namespaces/default
  ```
- **Validation**: Observe the serialized Protobuf representation of the `default` namespace stored at that key.

---

### Exercise 03: API Priority & Fairness Custom Concurrency Rules
- **Objective**: Create a custom FlowSchema that isolates heavy batch training queries from interfering with administrative commands.
- **Commands**:
  ```bash
  kubectl get flowschemas -o wide
  kubectl describe prioritylevelconfiguration workload-low
  ```
- **Validation**: Confirm that `workload-low` uses a dedicated concurrency share with queue size limits.

---

### Exercise 04: Production RBAC Role for an AI Engineering Team
- **Objective**: Grant an engineer permissions to launch training jobs and read logs, while blocking access to secret keys or cluster quotas.
- **Commands**:
  ```bash
  kubectl auth can-i create jobs -n k3s-alpha --as=ai-engineer
  kubectl auth can-i delete resourcequotas -n k3s-alpha --as=ai-engineer
  ```
- **Validation**: First command returns `yes`, second command returns `no`.

---

### Exercise 05: Validating Admission Webhook Policy Enforcement
- **Objective**: Test how an admission controller intercepts API requests before persistence.
- **Commands**:
  ```bash
  kubectl get validatingwebhookconfigurations
  ```
- **Validation**: Review active webhook configurations and confirm timeout parameters.

---

### Exercise 06: Tracing Controller Manager Reconciliation Cascades
- **Objective**: Observe the event cascade from Deployment $\to$ ReplicaSet $\to$ Pod $\to$ Scheduled Node.
- **Commands**:
  ```bash
  kubectl get events -n k3s-alpha --watch &
  kubectl create deployment cascade-test --image=nginx:alpine -n k3s-alpha
  ```
- **Validation**: Observe `ScalingReplicaSet`, `SuccessfulCreate`, `Scheduled`, and `Started` events in real time. Clean up: `kubectl delete deployment cascade-test -n k3s-alpha`.

---

### Exercise 07: Advanced GPU Scheduling: Taints & Tolerations
- **Objective**: Protect a GPU node from running non-GPU workloads.
- **Commands**:
  ```bash
  kubectl taint nodes --all accelerator=gpu:NoSchedule
  kubectl run cpu-test --image=alpine -- sleep 60
  kubectl get pod cpu-test
  ```
- **Validation**: Verify `cpu-test` is stuck in `Pending` due to `untolerated taint`. Clean up: `kubectl delete pod cpu-test` and remove taint with `-`.

---

### Exercise 08: Distributed Training Gang Scheduling with Kueue
- **Objective**: Understand all-or-nothing scheduling for distributed PyTorch.
- **Commands**:
  ```bash
  kubectl get clusterqueues,resourceflavors
  ```
- **Validation**: Verify that the cluster queue monitors total GPU availability before releasing jobs.

---

### Exercise 09: Tracing Virtual Ethernet (veth) Packet Flows on Host
- **Objective**: Match a Pod's `eth0` interface with its physical host peer interface.
- **Commands**:
  ```bash
  CONTAINER_ID=$(sudo crictl ps --name pytorch -q)
  PID=$(sudo crictl inspect $CONTAINER_ID | jq '.info.pid')
  sudo nsenter -t $PID -n ip link show eth0
  ```
- **Validation**: Find the peer interface index and match it against `ip link show` on the host OS.

---

### Exercise 10: iptables NAT Inspection for a ClusterIP Service
- **Objective**: Inspect the random probability balancing rules generated by `kube-proxy`.
- **Commands**:
  ```bash
  sudo iptables -t nat -L KUBE-SERVICES -n -v | head -n 30
  ```
- **Validation**: Locate the virtual ClusterIP address and trace the DNAT jump target.

---

### Exercise 11: CoreDNS Query Tracing & Fixing the ndots:5 AI Latency Bug
- **Objective**: Prove that external API requests execute 4 sequential queries under `ndots:5`.
- **Commands**:
  ```bash
  kubectl exec -it pytorch-benchmark -n k3s-alpha -- cat /etc/resolv.conf
  ```
- **Validation**: Identify `options ndots:5` and verify that domains with trailing dots bypass local search paths.

---

### Exercise 12: Ingress Layer 7 Path Routing with Real-Time Streaming
- **Objective**: Configure an Ingress route that disables proxy buffering for real-time LLM token streaming.
- **Commands**:
  ```bash
  kubectl get ingress -A
  ```
- **Validation**: Verify that `proxy-buffering: "off"` is present in the ingress annotations.

---

### Exercise 13: StatefulSet Vector Database with Deterministic Storage
- **Objective**: Deploy a 2-node vector database ensuring each replica gets an independent persistent volume.
- **Commands**:
  ```bash
  kubectl get statefulset -n k3s-alpha
  kubectl get pvc -n k3s-alpha -l app=vector-db
  ```
- **Validation**: Confirm PVC names match `data-vector-db-0` and `data-vector-db-1`.

---

### Exercise 14: Indexed Job for Sharded AI Dataset Preprocessing
- **Objective**: Run parallel data workers where each worker automatically knows its shard index.
- **Commands**:
  ```bash
  kubectl apply -f data-tokenizer-job.yaml
  kubectl logs -l job-name=dataset-tokenizer -n k3s-alpha --tail=20
  ```
- **Validation**: Confirm that logs from different pods show distinct `JOB_COMPLETION_INDEX` values (`0`, `1`, `2`, `3`).

---

### Exercise 15: High-Speed Local NVMe Storage with WaitForFirstConsumer
- **Objective**: Enforce that a PVC is only bound to physical storage once the Pod's GPU node is scheduled.
- **Commands**:
  ```bash
  kubectl get storageclass local-path -o yaml | grep volumeBindingMode
  ```
- **Validation**: Confirm output shows `volumeBindingMode: WaitForFirstConsumer`.

---

### Exercise 16: Enforcing Hard 5% Compute and Storage Quotas on a Tenant
- **Objective**: Verify that `k3s-alpha` cannot exceed 3.2 CPU cores, 6.4 GB RAM, and 50 GB storage.
- **Commands**:
  ```bash
  kubectl get resourcequota compute-quota-5pct -n k3s-alpha
  ```
- **Validation**: Inspect `USED` vs `HARD` limits. Confirm that any pod exceeding the limit is rejected with `exceeded quota`.

---

### Exercise 17: Generating and Validating Host CDI Device Profiles
- **Objective**: Interrogate the Container Device Interface (CDI) for NVIDIA accelerators.
- **Commands**:
  ```bash
  sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
  nvidia-ctk cdi list
  ```
- **Validation**: Confirm `nvidia.com/gpu=0` is registered.

---

### Exercise 18: Configuring NVIDIA Device Plugin with 10 GPU Time-Slices
- **Objective**: Divide 1 physical Blackwell GPU into 10 schedulable time-slices.
- **Commands**:
  ```bash
  kubectl get node -o jsonpath='{.items[0].status.allocatable.nvidia\.com/gpu}'
  ```
- **Validation**: Output returns `10`.

---

### Exercise 19: Running a Multi-Tenant PyTorch GPU Matrix Benchmark
- **Objective**: Execute a 10,000 x 10,000 matrix multiplication on the GPU from inside the container.
- **Commands**:
  ```bash
  kubectl logs pytorch-benchmark -n k3s-alpha
  ```
- **Validation**: Verify that output reports `100 MatMuls Completed in: X.XX seconds` on `NVIDIA Blackwell / GB10`.

---

### Exercise 20: Full Disaster Recovery: Restoring etcd from Snapshot
- **Objective**: Take a point-in-time snapshot and verify its integrity hash.
- **Commands**:
  ```bash
  ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-test.db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key
  
  ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-test.db -w table
  ```
- **Validation**: Output displays valid SHA-256 hash, revision number, and total keys stored.

---

🎉 **Mastery Complete!** You have now completed the entire 20-volume curriculum from Kubernetes core internals to enterprise NVIDIA AI supercomputing!
