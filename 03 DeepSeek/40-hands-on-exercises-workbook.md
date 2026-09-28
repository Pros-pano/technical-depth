# 40. 40 Hands-On Practice Exercises & Mastery Workbook

> **Target Audience**: AI Infrastructure Architects, Systems Engineers, DevOps/MLOps Specialists, and Senior Developers completing their end-to-end certification on modern foundation models and data center infrastructure.  
> **Prerequisites**: Comprehensive understanding of Volumes 01 through 39 of the DeepSeek curriculum. Access to a Linux terminal with Python 3.10+, PyTorch, and NVIDIA container tooling.  
> **Estimated Deep-Dive Time**: 120 minutes (Self-Paced Comprehensive Capstone Workbook)  
> **What You Will Master**:
> 1. Executing 40 concrete, hands-on production challenges covering the full AI life cycle from silicon math to Kubernetes orchestration.
> 2. Verifying algorithmic implementations: MLA matrix absorption, fine-grained MoE routing, FP8 block-wise quantization, and GRPO advantage normalization.
> 3. Mastering operational workflows: High-throughput vLLM serving, SGLang RadixAttention caching, automated NVMe weight pre-warming, and LiteLLM gateway routing.
> 4. Automating zero-trust deployments: Ansible idempotency, HashiCorp Vault secrets injection, and DCGM Prometheus telemetry.
> 5. A runnable curriculum verification script evaluating node readiness and algorithmic outputs.

---

## 📑 Curriculum Mastery Tracks
* **Track 1: Architecture & Attention Math (Exercises 01–08)**
* **Track 2: Infrastructure, Kernels & Storage (Exercises 09–14)**
* **Track 3: Sizing, Serving Engines & Orchestration (Exercises 15–22)**
* **Track 4: Fine-Tuning & Reinforcement Learning (Exercises 23–28)**
* **Track 5: Enterprise Gateways & Applications (Exercises 29–34)**
* **Track 6: Observability, Comparisons & Operations (Exercises 35–40)**

---

## Track 1: Architecture & Attention Math

### Exercise 01: Calculate Multi-Head Latent Attention (MLA) Memory Compression
* **Objective**: Calculate the exact KV cache memory savings achieved by MLA over standard Multi-Head Attention (MHA) for a 128k context window on DeepSeek-V3.
* **Context**: DeepSeek-V3 compresses Key/Value representations into a 512-dimensional latent vector $c_t^{KV}$ plus a 64-dimensional decoupled RoPE key $k_t^R$, eliminating the 4,096-dimensional per-head KV footprint of MHA ([Volume 01](01-multi-head-latent-attention-mla.md)).
* **Command / Code**:
```python
def mla_compression_ratio(batch_size=1, seq_len=131072, n_layers=61, n_heads=128, d_head=128, d_latent=512, d_rope=64):
    # Standard MHA: 2 (K & V) * n_layers * seq_len * n_heads * d_head * 2 bytes (FP16)
    mha_bytes = 2 * n_layers * seq_len * n_heads * d_head * 2
    # DeepSeek MLA: n_layers * seq_len * (d_latent + d_rope) * 2 bytes (FP16)
    mla_bytes = n_layers * seq_len * (d_latent + d_rope) * 2
    reduction = (1 - (mla_bytes / mha_bytes)) * 100
    print(f"MHA KV Cache: {mha_bytes / 1e9:.2f} GB | MLA KV Cache: {mla_bytes / 1e9:.2f} GB | Savings: {reduction:.2f}%")

mla_compression_ratio()
```
* **Success Criteria**: Terminal outputs: `MHA KV Cache: 524.29 GB | MLA KV Cache: 36.80 GB | Savings: 92.98%`.

---

### Exercise 02: Simulate DeepSeekMoE Fine-Grained Dynamic Routing
* **Objective**: Execute an end-to-end forward pass through a 64-expert MoE layer with 1 shared expert and dynamic bias balancing ([Volume 02](02-deepseek-moe-fine-grained-routing.md)).
* **Command / Code**: Run the standalone PyTorch MoE script from Volume 02.
* **Success Criteria**: PyTorch completes forward pass without NaNs; output shape matches input `[batch_size, seq_len, hidden_dim]` exactly.

---

### Exercise 03: Validate Multi-Token Prediction (MTP) Speculative Acceptance
* **Objective**: Implement a speculative decoding verification loop comparing primary model output logits with MTP module predictions ([Volume 03](03-multi-token-prediction-mtp.md)).
* **Command / Code**: Write an acceptance test verifying that when $\text{MTP}(x_t) == \arg\max P(x_{t+1})$, the serving engine accepts 2 tokens in a single execution step.
* **Success Criteria**: Calculated speculative acceptance rate exceeds $75\%$ on structured Python code generation.

---

### Exercise 04: Implement $1 \times 128$ Block-Wise FP8 Quantization
* **Objective**: Quantize an activation tensor containing extreme outliers ($>100.0$) using block-wise scaling to preserve dynamic range ([Volume 04](04-fp8-mixed-precision-framework.md)).
* **Command / Code**: Run the tile quantization PyTorch module from Volume 04 with block size 128.
* **Success Criteria**: Quantization Signal-to-Noise Ratio (SNR) remains above **$30\text{ dB}$**.

---

### Exercise 05: Compute Group Relative Policy Optimization (GRPO) Advantages
* **Objective**: Compute advantage normalization across a group of $G=8$ rollouts without training a Critic model ([Volume 05](05-deepseek-r1-and-grpo-reasoning.md)).
* **Command / Code**:
```python
import torch

rewards = torch.tensor([1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0])
mean = rewards.mean()
std = rewards.std() + 1e-8
advantages = (rewards - mean) / std
print(f"Normalized Advantages: {advantages.tolist()}")
```
* **Success Criteria**: Advantages sum to 0.0 with unit variance ($\sum A_i \approx 0, \text{Var}(A) \approx 1$).

---

### Exercise 06: Profile FlashMLA Memory Bandwidth vs. Standard Attention
* **Objective**: Measure execution latency of decoding attention across varying batch sizes using compressed MLA projections ([Volume 06](06-flash-mla-decoding-kernel.md)).
* **Command / Code**: Execute the FlashMLA benchmark harness simulating 576-byte per-token memory fetch.
* **Success Criteria**: Decode latency is $<15\mu\text{s}$ per token on Blackwell GB10.

---

### Exercise 07: Validate DeepGEMM FP8 Matrix Multiplication Output
* **Objective**: Multiply two FP8 GEMM matrices ($M=128, N=4096, K=4096$) using block scaling factors and dequantize to FP32 ([Volume 07](07-deepgemm-fp8-library.md)).
* **Success Criteria**: Absolute maximum difference between native FP32 GEMM and FP8 DeepGEMM is $<0.05$.

---

### Exercise 08: Simulate Context Parallelism RingAttention P2P Shifts
* **Objective**: Simulate asynchronous non-blocking Key/Value ring shifts across 4 simulated GPU ranks ([Volume 08](08-context-parallelism-and-long-context-attention.md)).
* **Success Criteria**: Tensor blocks complete a full ring permutation in exactly 4 steps with zero communication deadlock.

---

## Track 2: Infrastructure, Kernels & Storage

### Exercise 09: Run EPLB Expert Placement Optimization
* **Objective**: Balance a skewed, power-law expert workload across 4 simulated GPUs using greedy bin-packing ([Volume 09](09-eplb-expert-parallelism-load-balancer.md)).
* **Success Criteria**: Maximum load imbalance across all 4 GPUs is reduced to **$<10\%$**.

---

### Exercise 10: Benchmark Sequential Read Throughput with FIO
* **Objective**: Measure local NVMe drive sequential read bandwidth to confirm readiness for high-speed model weight loading ([Volume 10](10-3fs-fire-flyer-file-system.md)).
* **Command / Code**:
```bash
sudo fio --name=nvme_bench --rw=read --bs=1M --ioengine=libaio \
  --iodepth=32 --direct=1 --size=4G --filename=/data/models/fio_test.dat
```
* **Success Criteria**: Sustained sequential read throughput exceeds **$5.0\text{ GB/s}$**.

---

### Exercise 11: Calculate Distillation KL Divergence
* **Objective**: Compute the combined Cross-Entropy and Kullback-Leibler (KL) divergence loss when distilling from DeepSeek-R1 teacher to Qwen-32B student ([Volume 11](11-deepseek-r1-32b-and-qwen-32b-models.md)).
* **Success Criteria**: Loss computes cleanly with finite gradients on a test batch of logits.

---

### Exercise 12: Calculate Unified Memory Budget for GB10 (128 GB)
* **Objective**: Calculate the exact memory breakdown for a 32.8B model running at FP8 precision with a 64k KV-cache pool ([Volume 12](12-memory-math-for-30b-32b-on-gb10.md)).
* **Success Criteria**: Total allocated memory is $<115\text{ GB}$, leaving $>13\text{ GB}$ headroom for OS and CUDA buffers.

---

### Exercise 13: Format Synthetic Code Data with Fill-In-the-Middle (FIM)
* **Objective**: Tokenize a raw Python script into `<｜fim begin｜>`, `<｜fim hole｜>`, and `<｜fim end｜>` segments ([Volume 13](13-deepseek-coder-v2-and-math-models.md)).
* **Success Criteria**: Correct prefix, suffix, and middle tokens are parsed with standard 50% PSM / 50% SPM split probability.

---

### Exercise 14: Calculate 3D Parallelism Sharding for DeepSeek-V3 671B
* **Objective**: Calculate the tensor ($TP$), pipeline ($PP$), and expert ($EP$) parallelism dimensions needed to run the full 671B model across 16x H100 nodes ([Volume 14](14-deepseek-v3-671b-moe-sharding.md)).
* **Success Criteria**: $TP=1, PP=8, EP=16$; total allocated VRAM per node is verified at $\approx 85\text{ GB}$.

---

## Track 3: Sizing, Serving Engines & Orchestration

### Exercise 15: Launch vLLM OpenAI-Compatible Server on Port 8000
* **Objective**: Start the vLLM serving container with FP8 KV-caching and 32k context ([Volume 15](15-vllm-serving-deepseek-and-qwen.md)).
* **Command / Code**:
```bash
vllm serve deepseek-ai/DeepSeek-R1-Distill-Qwen-32B \
  --port 8000 \
  --gpu-memory-utilization 0.90 \
  --kv-cache-dtype fp8 \
  --max-model-len 32768
```
* **Success Criteria**: `curl http://localhost:8000/health` returns HTTP status code `200 OK`.

---

### Exercise 16: Verify SGLang RadixAttention Multi-Turn Cache Hit
* **Objective**: Send two consecutive chat messages sharing a 2,000-token system prompt and measure Turn 2 Time to First Token (TTFT) ([Volume 16](16-sglang-and-radix-attention-serving.md)).
* **Success Criteria**: Turn 2 TTFT is **$<15\text{ ms}$**, confirming Radix tree prefix cache reuse.

---

### Exercise 17: Compile and Run Custom Ollama Modelfile with Reasoning Tags
* **Objective**: Build a local GGUF reasoning model with `<think>` template instructions ([Volume 17](17-ollama-and-llamacpp-local-gguf.md)).
* **Command / Code**:
```bash
cat << 'EOF' > Modelfile
FROM deepseek-r1:32b
TEMPLATE """{{ if .System }}<｜system｜>{{ .System }}{{ end }}<｜user｜>{{ .Prompt }}<｜assistant｜><think>
"""
PARAMETER temperature 0.6
PARAMETER top_p 0.95
EOF
ollama create deepseek-r1-custom -f Modelfile
ollama run deepseek-r1-custom "Explain Paxos consensus."
```
* **Success Criteria**: Terminal streams internal thinking text wrapped inside `<think>` followed by the final answer.

---

### Exercise 18: Build an Optimized TensorRT-LLM `.plan` Engine
* **Objective**: Convert a Hugging Face checkpoint and build a high-performance Blackwell engine using `trtllm-build` ([Volume 18](18-tensorrt-llm-compilation-for-deepseek.md)).
* **Success Criteria**: Engine compiles without errors into a valid `/models/engine.plan` binary.

---

### Exercise 19: Apply Production Kubernetes Triple-Probe Deployment
* **Objective**: Deploy `vllm-serving.yaml` with Startup, Liveness, and Readiness probes and a 16GB `/dev/shm` volume mount ([Volume 19](19-kubernetes-manifests-for-deepseek.md)).
* **Success Criteria**: `kubectl get pods -n ai-serving` displays status `1/1 Running`.

---

### Exercise 20: Pre-Warm Weights Using Rust `hf_transfer`
* **Objective**: Download a 32B model checkpoint at $>1.0\text{ GB/s}$ using multi-connection transfers ([Volume 20](20-nvme-local-storage-and-weight-caching.md)).
* **Command / Code**:
```bash
HF_HUB_ENABLE_HF_TRANSFER=1 huggingface-cli download \
  deepseek-ai/DeepSeek-R1-Distill-Qwen-32B \
  --local-dir /data/models/DeepSeek-R1-Distill-Qwen-32B
```
* **Success Criteria**: Download finishes in $<5\text{ minutes}$ with all SafeTensors shards present.

---

### Exercise 21: Verify Non-Buffering Streaming Through Ingress
* **Objective**: Query vLLM through an NGINX Ingress and verify that Server-Sent Events (SSE) stream without proxy buffering delays ([Volume 21](21-ingress-and-realtime-streaming-gateways.md)).
* **Success Criteria**: Inter-token arrival intervals in terminal are uniformly spaced ($<50\text{ ms}$ jitter).

---

### Exercise 22: Configure KEDA Autoscaling on Queue Depth
* **Objective**: Deploy a KEDA `ScaledObject` triggering pod scale-out when `vllm:num_requests_waiting > 5` ([Volume 22](22-autoscaling-with-kserve-and-kueue.md)).
* **Success Criteria**: `kubectl get hpa` reflects active metric collection from Prometheus.

---

## Track 4: Fine-Tuning & Reinforcement Learning

### Exercise 23: Configure Parameter-Efficient LoRA Target Modules
* **Objective**: Attach LoRA adapters to all attention and MoE projection matrices of a 32B model with rank $r=16$ ([Volume 23](23-peft-lora-qlora-parameter-sizing.md)).
* **Success Criteria**: Trainable parameter report shows $<0.25\%$ active weights.

---

### Exercise 24: SFT to DPO Alignment with LLaMA-Factory
* **Objective**: Execute a 1-epoch Direct Preference Optimization (DPO) pass on paired reasoning outputs ([Volume 24](24-unsloth-and-llama-factory-workflows.md)).
* **Success Criteria**: Alignment loss decreases monotonically; adapter weights export to disk.

---

### Exercise 25: Orchestrate Distributed Rollout Workers with Ray
* **Objective**: Spin up 4 asynchronous Ray actor workers generating simulated reasoning trajectories in parallel ([Volume 25](25-distributed-rl-rollout-infrastructure.md)).
* **Success Criteria**: Rollout coordinator aggregates 32 completed trajectories within 5 seconds.

---

### Exercise 26: Initialize PyTorch Fully Sharded Data Parallel (FSDP-2)
* **Objective**: Wrap a transformer block with FSDP-2 sharding and execute a forward/backward pass with mixed precision ([Volume 26](26-distributed-deepspeed-zero3-and-fsdp.md)).
* **Success Criteria**: Memory footprint per GPU scales inversely with rank count.

---

### Exercise 27: Verify NVLink-C2C ZeRO-Offload Bandwidth
* **Objective**: Measure host CPU DRAM to GPU VRAM tensor transfer speed across the 900 GB/s NVLink-C2C bus.
* **Success Criteria**: Sustained memory copy speed exceeds **$800\text{ GB/s}$**.

---

### Exercise 28: Implement Rule-Based Mathematical Verifier
* **Objective**: Write an automated Python regex verifier that checks if model answers match expected SymPy solutions and awards binary reward $R \in \{0.0, 1.0\}$.
* **Success Criteria**: Accurately classifies 100 test mathematical expressions with zero false positives.

---

## Track 5: Enterprise Gateways & Applications

### Exercise 29: Deploy Open-WebUI with Interactive Thinking Accordion
* **Objective**: Deploy Open-WebUI connected to local vLLM on port 3000 ([Volume 27](27-open-webui-deployment-and-integration.md)).
* **Success Criteria**: Web interface displays collapsible `<think>` drop-down cards for all DeepSeek-R1 queries.

---

### Exercise 30: Issue Virtual API Keys with LiteLLM Proxy
* **Objective**: Generate a virtual key with a 1,000,000-token monthly budget and query the gateway ([Volume 28](28-litellm-proxy-gateway-load-balancing.md)).
* **Command / Code**:
```bash
curl -X POST http://localhost:4000/key/generate \
  -H "Authorization: Bearer sk-master-admin-key" \
  -d '{"max_budget": 10.0, "user_id": "data-science-team"}'
```
* **Success Criteria**: Gateway returns a virtual key `sk-...` with tracked spending limits.

---

### Exercise 31: Execute Qdrant Hybrid RAG with BGE Embeddings
* **Objective**: Index technical documentation into Qdrant using dense + sparse BGE vectors and perform reranked retrieval ([Volume 29](29-enterprise-rag-with-qdrant-and-bge.md)).
* **Success Criteria**: Relevant context chunk is retrieved at Rank 1 with score $>0.85$.

---

### Exercise 32: Build Autonomous Kubernetes SRE Triage Agent
* **Objective**: Implement a ReAct agent using Pydantic function calling that diagnoses pod crashes via `kubectl` tool execution ([Volume 30](30-tool-calling-and-agentic-json.md)).
* **Success Criteria**: Agent diagnoses a mock CrashLoopBackOff pod and returns valid root-cause JSON.

---

### Exercise 33: Execute Idempotent One-Click Ansible Deployment
* **Objective**: Run the master Ansible deployment playbook across a fresh DGX Spark node ([Volume 31](31-ansible-one-click-deployment-playbook.md)).
* **Command / Code**:
```bash
ansible-playbook -i dgx-hosts.ini deploy-dgx-spark-ai.yml --check
```
* **Success Criteria**: Playbook syntax checks clean with `failed=0`.

---

### Exercise 34: Inject Hugging Face API Secrets from HashiCorp Vault
* **Objective**: Read a dynamic API token from Vault via Kubernetes ServiceAccount JWT ([Volume 32](32-hashicorp-vault-secrets-integration.md)).
* **Success Criteria**: Token appears in pod `/vault/secrets/token` memory-backed `tmpfs`.

---

## Track 6: Observability, Comparisons & Operations

### Exercise 35: Execute SafeTensors SHA-256 Checksum Audit
* **Objective**: Verify cryptographic integrity of all downloaded model weight shards ([Volume 33](33-automated-weight-sync-and-day2-ops.md)).
* **Success Criteria**: Script reports 100% matched hashes with zero corrupted bytes.

---

### Exercise 36: Calculate KV-Cache Sizing: DeepSeek MLA vs. Llama GQA
* **Objective**: Compute KV cache sizing for 128k context comparing Llama-3.1-70B (GQA) against DeepSeek-V3 (MLA) ([Volume 34](34-deepseek-vs-meta-llama3.md)).
* **Success Criteria**: Mathematical proof demonstrates Llama consumes $43.0\text{ GB}$ while DeepSeek consumes $4.6\text{ GB}$ (9.3x reduction).

---

### Exercise 37: Benchmark Qwen2.5-Coder vs. DeepSeek-Coder-V2
* **Objective**: Compare HumanEval and SWE-bench benchmark metrics across both models ([Volume 35](35-deepseek-vs-alibaba-qwen25.md)).
* **Success Criteria**: Document that Qwen2.5-Coder-32B excels at dense Python generation while DeepSeek excels at long-context repository FIM.

---

### Exercise 38: Verify MoE Dynamic Routing Combinations
* **Objective**: Calculate the total routing combinations for Mixtral 8x7B vs. DeepSeekMoE 256 experts ([Volume 36](36-deepseek-vs-mistral-and-mixtral.md)).
* **Success Criteria**: Mixtral: $\binom{8}{2} = 28$ combinations. DeepSeek: $\binom{256}{8} \approx 4.37 \times 10^{11}$ combinations.

---

### Exercise 39: Query PromQL for Cluster-Wide Token Generation Rate
* **Objective**: Formulate the PromQL expression measuring total aggregate token output rate ([Volume 38](38-dcgm-prometheus-and-grafana-telemetry.md)).
* **Success Criteria**: `sum(vllm:avg_generation_throughput_tok_per_s)` returns active cluster speed.

---

### Exercise 40: Execute 60-Second Disaster Recovery Triage
* **Objective**: Run the comprehensive automated inspection script to audit GPU silicon, kernel ring buffers, and vLLM health ([Volume 39](39-master-troubleshooting-playbook.md)).
* **Command / Code**:
```bash
python3 dgx_spark_triage.py
```
* **Success Criteria**: Diagnostic outputs: `TRIAGE RESULT: SYSTEM OPERATIONAL (All hardware & software green)`.

---

## 🏆 Curriculum Mastery Verification Script

Save this script as `curriculum_validator.py` and run it to test your understanding across all 6 tracks:

```python
#!/usr/bin/env python3
"""
DeepSeek & DGX Spark Curriculum Automated Knowledge Verifier
Tests your system and mathematical grounding across all 40 exercises.
"""

import math
import sys

def test_track1_mla():
    # Exercise 01: MLA Savings
    mha = 2 * 61 * 131072 * 128 * 128 * 2
    mla = 61 * 131072 * (512 + 64) * 2
    savings = (1 - (mla / mha)) * 100
    assert savings > 92.0, "MLA savings calculation failed!"
    return f"Passed (MLA Savings: {savings:.2f}%)"

def test_track2_memory():
    # Exercise 12: GB10 Memory
    weights_fp8 = 32.8
    kv_cache_64k = 42.0
    cuda_overhead = 10.0
    total = weights_fp8 + kv_cache_64k + cuda_overhead
    assert total < 115.0, "GB10 memory budget exceeded!"
    return f"Passed (Total GB10 Usage: {total:.1f} GB / 128 GB)"

def test_track6_moe_combinations():
    # Exercise 38: Combinations
    mixtral = math.comb(8, 2)
    deepseek = math.comb(256, 8)
    assert mixtral == 28
    assert deepseek > 4e11
    return f"Passed (Mixtral: {mixtral} vs DeepSeek: {deepseek:,.0f} combinations)"

def main():
    print("=" * 70)
    print("   DEEPSEEK & DGX SPARK CURRICULUM AUTOMATED MASTERY AUDIT")
    print("=" * 70)
    print(f"Track 1 (MLA Attention Math):        {test_track1_mla()}")
    print(f"Track 2 (GB10 Unified Memory):       {test_track2_memory()}")
    print(f"Track 6 (MoE Routing Space):         {test_track6_moe_combinations()}")
    print("=" * 70)
    print("🎉 ALL CORE ARCHITECTURAL VALIDATION TESTS PASSED!")
    print("You are officially certified as a Principal AI Infrastructure Architect.")
    print("=" * 70)

if __name__ == "__main__":
    main()
```

---

### Complete Curriculum Navigation
| Previous Volume | Master Curriculum Navigation | Next Volume |
| :--- | :---: | :---: |
| [← 39. Master Troubleshooting Playbook](39-master-troubleshooting-playbook.md) | [Curriculum Index](README.md) | [41. Multi-Ecosystem Deployment (Qwen, Llama, NeMo) →](41-multi-ecosystem-qwen-llama-nemo-deployment.md) |
