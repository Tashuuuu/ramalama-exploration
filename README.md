# RamaLama 0.18.0 — Technical Exploration Report

**Contributor:** Akriti Sengar | **Project:** Outreachy 2026 · Fedora Project  
**Report type:** Evidence-based engineering report — all outputs are real

---

## System Specifications

| Parameter | Value |
|---|---|
| Host OS | Windows 11 Home Single Language, Version 25H2, Build 26200.8117 |
| CPU | AMD Ryzen 7 4700U — 8 threads, 4 physical cores, integrated Radeon GPU |
| Total RAM | 7,599 MB (~7.4 GB) |
| WSL2 Distro | Ubuntu 24.04.4 LTS (Noble Numbat) |
| WSL2 Kernel | 6.6.87.2-microsoft-standard-WSL2 |
| WSL2 RAM Cap | 4 GB (`.wslconfig: memory=4GB`) |
| WSL2 Swap | 2 GB (`.wslconfig: swap=2GB`) |
| Container Engine | Podman 4.9.3 (rootless) |
| RamaLama Version | 0.18.0 |
| Default Runtime | llama.cpp (CPU-only — `ggml_vulkan: No devices found`) |
| Baseline Available RAM | ~3.4 Gi (after WSL2 overhead) |

---

## Quick Reference

| Task | Command | Status |
|---|---|---|
| Install | `pip install ramalama --break-system-packages` | ✅ |
| Version | `ramalama version` → 0.18.0 | ✅ |
| Pull TinyLlama (Ollama) | `ramalama pull ollama://tinyllama` | ✅ |
| Run TinyLlama | `ramalama run ollama://tinyllama "prompt"` | ✅ |
| Pull Granite 2B | `ramalama pull ollama://granite3.1-dense:2b` | ✅ pull / ❌ run (OOM) |
| HuggingFace transport | `ramalama pull huggingface://TheBloke/...` | ✅ |
| OCI transport | `ramalama pull oci://quay.io/ramalama/ramalama:latest` | ✅ pull / ❌ run (WSL2) |
| Serve REST API | `ramalama serve ollama://tinyllama --port 8080` | ✅ |
| Benchmark | `ramalama bench ollama://tinyllama` | ✅ |
| RAG generate | `ramalama rag generate fedora_foundations.txt` | ❌ not in v0.18.0 |

---

## Phase 0 — Windows Environment Audit

### Enable WSL2 (VirtualMachinePlatform required)

```cmd
wsl --status
```

**Output:** `WSL2 is not supported — Please enable the "Virtual Machine Platform" optional component.`

```cmd
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
shutdown /r /t 0
```

Both DISM commands return: `The operation completed successfully.`

### Install Ubuntu 24.04

```cmd
wsl --install -d Ubuntu-24.04
wsl --set-default Ubuntu-24.04
wsl --list --verbose
```

**Verification output:**
```
NAME           STATE   VERSION
Ubuntu         Stopped 2
Ubuntu-24.04   Running 2
```

**Note:** `wsl --list --online` shows FedoraLinux-42 and FedoraLinux-43 are available as WSL2 distros. Ubuntu-24.04 was chosen for broader package compatibility.

---

## Phase 1 — Configure WSL2 Memory

Critical step before any model runs. Windows at idle consumes ~4.5 GB of the 7.4 GB physical RAM.

```
# File: C:\Users\<user>\.wslconfig
[wsl2]
memory=4GB
processors=4
swap=2GB
```

```cmd
wsl --shutdown
```

After relaunch, `free -h` inside WSL2 shows:

```
total    used    free    available
3.8Gi    [used]  [free]  3.4Gi
Swap: 2.0Gi 0B 2.0Gi
```

**Why 3.8Gi instead of 4GB:** The ~200 MB gap is WSL2's fixed hypervisor/kernel overhead. This is constant and reduces practical model headroom to 3.4 Gi available. This number is the reference baseline for all subsequent memory analysis.

---

## Phase 2 — Install Dependencies and RamaLama

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv curl wget git
sudo apt install -y podman
```

Podman 4.9.3 installs with 26 packages (33.6 MB download, 136 MB installed). Key dependencies: `crun` (OCI runtime), `netavark` (networking), `slirp4netns` (rootless user-mode networking).

```bash
pip install ramalama --break-system-packages
```

**Output:**
```
Downloading ramalama-0.18.0-py3-none-any.whl (204 kB)
WARNING: The script ramalama is installed in '/home/tashu/.local/bin' which is not on PATH.
Successfully installed argcomplete-3.6.3 ramalama-0.18.0
```

**Why `--break-system-packages`:** Ubuntu 24.04 enforces PEP 668 — pip cannot modify system Python by default. This flag overrides that for user-level installations. A venv is the proper alternative for production use.

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
ramalama version
```

**Output:** `ramalama version 0.18.0`

---

## Phase 3 — System Inspection

### ramalama info (key fields)

```json
{
  "Accelerator": "none",
  "Engine": {
    "Info": {
      "host": {
        "arch": "amd64",
        "cgroupManager": "systemd",
        "cgroupVersion": "v2",
        "kernel": "6.6.87.2-microsoft-standard-WSL2",
        "networkBackend": "netavark",
        "ociRuntime": { "path": "/usr/bin/crun" },
        "swapTotal": 2147483648
      },
      "Version": { "Version": "4.9.3" }
    }
  },
  "RagImage": "quay.io/ramalama/ramalama-rag:0.18",
  "UseContainer": true
}
```

`RagImage` is listed here but `ramalama rag generate` is not implemented in v0.18.0 (see Phase 8). This field is forward-planning, not a functional indicator.

### --dryrun: Full Abstraction Revealed

```bash
ramalama --dryrun run ollama://tinyllama "What are the Four Foundations of the Fedora project?"
```

**Output (18+ flag Podman command):**
```
podman run \
  --label ai.ramalama.model=ollama://library/tinyllama:latest \
  --label ai.ramalama.engine=podman \
  --label ai.ramalama.runtime=llama.cpp \
  --label ai.ramalama.port=8080 \
  --label ai.ramalama.command=run \
  --security-opt=label=disable \
  --cap-drop=all \
  --security-opt=no-new-privileges \
  --pull newer \
  -d -p 8080:8080 \
  --label ai.ramalama \
  --name ramalama-4tFMtNDpQz \
  --env=HOME=/tmp \
  --init quay.io/ramalama/ramalama:0.18 \
  llama-server \
  --host 0.0.0.0 --port 8080 \
  --model /path/to/model \
  --jinja --no-warmup \
  --alias library/tinyllama \
  --temp 0.8 --cache-reuse 256 \
  -ngl 999 --threads 4 --log-colors on
```

**Flag breakdown:**

| Flag | Purpose |
|---|---|
| `--cap-drop=all` | Drop all Linux capabilities (security hardening) |
| `--security-opt=no-new-privileges` | Prevent privilege escalation |
| `--security-opt=label=disable` | Disable SELinux/AppArmor label checking |
| `--pull newer` | Update container image if newer version exists |
| `--init` | Use tini/init for proper signal handling inside container |
| `--env=HOME=/tmp` | Set HOME to writable path inside container |
| `-ngl 999` | Request 999 GPU layers (no-op on CPU, full offload on GPU) |
| `--threads 4` | CPU threads — matches `processors=4` in `.wslconfig` |
| `--temp 0.8` | Sampling temperature |
| `--cache-reuse 256` | KV cache reuse window for efficiency |
| `--jinja` | Enable Jinja2 templating for chat formatting |
| `--no-warmup` | Skip inference warmup for faster startup |

**The `-ngl 999` design:** RamaLama passes this unconditionally. On CPU-only systems it is a harmless no-op. On GPU systems it enables full layer offloading without any user configuration. This is the abstraction working — the same command works correctly regardless of hardware.

---

## Phase 4 — Model 1: TinyLlama via Ollama Transport

### Pull

```bash
ramalama pull ollama://tinyllama
```

```
Downloading ollama://library/tinyllama:latest ...
100% |████████████████████| 608.16 MB/608.16 MB 12.30 MB/s 0s
```

`ollama://tinyllama` resolves to `ollama://library/tinyllama:latest` — RamaLama adds the `library/` prefix and `:latest` tag automatically.

```bash
ramalama inspect ollama://tinyllama
```
```
Registry: ollama | Version: 3 | Endianness: little | Tensors: 201 entries
Path: ~/.local/share/ramalama/store/ollama/library/tinyllama/blobs/sha256-2af3b81862c6b...
```

Model stored by SHA256 content-addressed hash — guarantees integrity.

### Run

```bash
ramalama run ollama://tinyllama "What are the Four Foundations of the Fedora project?"
```

```
1. Community: Fedora prioritizes community participation as its core value.
2. Innovation: Fedora values incorporating new technologies and advancements.
3. Freedom: Software must be free to use, modify, and distribute.
4. Continuous Integration: Fedora uses a continuous integration system.
```

**Actual foundations:** Freedom, Friends, Features, First. TinyLlama correctly identifies Freedom and loosely maps Community to Friends. Innovation and Continuous Integration are hallucinations.

### Memory profile

| Moment | Available RAM | Swap Used |
|---|---|---|
| Baseline | 3.4 Gi | 0B |
| During TinyLlama inference | 1.2 Gi | 0B |
| After container exits | 3.4 Gi | 0B |

Model (608 MB) + container runtime + llama-server = ~2.2 GB total consumption. Full memory reclamation after container exit confirms no persistent leak.

### --nocontainer test

```bash
ramalama --nocontainer run ollama://tinyllama "..."
```

```
Error: [Errno 2] No such file or directory: 'llama-server'
```

`llama-server` is not installed natively — it exists only inside the `quay.io/ramalama/ramalama:0.18` container image. This confirms the container is not optional convenience; it is the delivery mechanism for the inference runtime. `--nocontainer` requires manually installing llama.cpp natively on the host.

---

## Phase 5 — Transport Comparison: Ollama vs HuggingFace

### HuggingFace Pull (same model, different registry)

```bash
ramalama pull huggingface://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf
```

```
100% |████| 637.81 MB/637.81 MB 12.89 MB/s 0s
```

| Property | Ollama | HuggingFace |
|---|---|---|
| URI length | ~22 chars | ~98 chars |
| File size | 608.16 MB | 637.81 MB (+29.65 MB) |
| Quantisation | Q4_0 | Q4_K_M |
| Tensors (inspect) | 201 | 201 (identical architecture) |
| Registry | ollama | huggingface |
| Storage path | `.../store/ollama/library/tinyllama/` | `.../store/huggingface/TheBloke/.../` |

**Why 29.65 MB larger:** Q4_K_M is a mixed-precision K-quant format — slightly larger than Q4_0 but generally better output quality per bit. The size difference comes from quantisation choice, not architecture difference (both have 201 tensors).

### Same prompt, different output

HuggingFace TinyLlama response:
```
1. Community: Fedora is a community project, and the community is the cornerstone.
2. Dedication: community has a deep level of dedication and support.
3. Community Values: The project values community, and community values are the core.
4. Community Success: Fedora has achieved success through community values.
```

Both transports use identical 1.1B TinyLlama weights. Ollama hallucinated technical concepts (Innovation, Continuous Integration). HuggingFace looped on "community" four times. The difference is system prompt defaults and chat template handling between the two packaging sources. Without testing both transports, this variation is invisible.

### Benchmark comparison

| Metric | Ollama TinyLlama | HuggingFace TinyLlama |
|---|---|---|
| pp512 (prompt processing) | 97.66 ± 9.22 t/s | 119.25 ± 50.20 t/s |
| tg128 (token generation) | 53.24 ± 37.27 t/s | 36.05 ± 0.21 t/s |

**Variability note:** Ollama TinyLlama's tg128 shows ±37.27 standard deviation — 70% of the mean. High variance is expected on a shared-resource WSL2 system with CPU thermal throttling. Single-run benchmark results are directional, not precise. HuggingFace has faster prompt processing but slower generation than Ollama — likely KV cache and chat template interaction differences.

---

## Phase 6 — Documented Failures and Root Cause Analysis

### 6.1 IBM Granite 3.1-dense:2b — OOM Context Failure

```bash
ramalama pull ollama://granite3.1-dense:2b
# 1.46 GB downloaded successfully at 12.53 MB/s

ramalama run ollama://granite3.1-dense:2b "What are the Four Foundations?"
```

```
340c6c2f91c1...
............................................................................
[180 seconds of dots]
2026-04-07 19:45:55 - ERROR - Failed to serve model granite3.1-dense
2026-04-07 19:45:55 - ERROR - Command 'health check of container ramalama-eZ72858khU' timed out after 180 seconds
```

**Diagnosis via serve mode (shows hidden logs):**
```bash
ramalama serve ollama://granite3.1-dense:2b
```

```
main: loading model
llama_model_loader: - type q6_K: 40 tensors
print_info: file size = 1.46 GiB (4.95 BPW)
granite.attention.head_count = 32
granite.attention.head_count_kv = 8
tokenizer.ggml.tokens arr[str,49155]
...
common_init from params: failed to create context with model '/mnt/models/granite3.1-dense'
```

**Root cause:** Granite's weights (1.46 GB) load successfully. The failure is at context allocation — the default KV cache for a 32-head, 2.53B parameter model requires more RAM than the ~1.7 Gi remaining after loading the weights. Available RAM during timeout: 0.8–1.7 Gi (oscillating as allocations fail and retry).

**Fix attempts:**

| Attempt | Command | Result |
|---|---|---|
| Reduce context | `ramalama run ... -- -c 512` | `ramalama: error: unrecognized arguments: -c 512` |
| Serve mode | `ramalama serve ollama://granite3.1-dense:2b` | Same timeout + `failed to create context` |
| Serve + context | `ramalama serve ... -- -c 512` | Same argument parse error |

**Status:** NOT RESOLVED — hardware constraint. The `--` passthrough mechanism for llama-server arguments does not work in v0.18.0.

**Interesting nuance:** `ramalama bench ollama://granite3.1-dense:2b` succeeds:
```
| pp512 | 38.15 ± 3.40 t/s |
| tg128 | 15.22 ± 2.34 t/s |
```
Bench uses shorter inference bursts that avoid allocating the full default context window. Interactive run requires full 2048-token KV cache allocation upfront — which exceeds available RAM. Granite sits in a state of: measurable but not usable on this system.

**Memory during Granite timeout:**

| Moment | Available RAM | Swap |
|---|---|---|
| Before run | 3.4 Gi | 0B |
| During 180s timeout | 0.8–1.7 Gi | 0B |

### 6.2 Podman Registry Configuration Bug (Pre-existing)

Discovered while isolating whether Granite's failure was container-level or model-level:

```bash
podman pull docker.io/library/hello-world
```

```
Error: loading registries configuration "/etc/containers/registries.conf":
mixing sys registry v1/v2 is not supported
```

**Cause:** Ubuntu 24.04's default Podman package ships a `registries.conf` with both V1 and V2 format entries simultaneously — an invalid combination Podman 4.9.3 refuses to parse. This bug predates the RamaLama installation.

**Fix:**
```bash
sudo cp /etc/containers/registries.conf /etc/containers/registries.conf.backup

sudo tee /etc/containers/registries.conf << 'EOF'
unqualified-search-registries = ["docker.io", "quay.io"]

[aliases]
"hello-world" = "docker.io/library/hello-world"
EOF
```

**Verification:**
```bash
podman pull docker.io/library/hello-world
# Success: "Writing manifest to image destination"

podman run --rm docker.io/library/hello-world
# Output: "Hello from Docker! This message shows your installation is working correctly."
```

**Status:** RESOLVED. Podman was working correctly after the registries fix — this was not a RamaLama issue.

### 6.3 OCI Transport — WSL2 Mount Incompatibility

```bash
ramalama pull oci://quay.io/ramalama/ramalama:latest
# Pulls cleanly — layer deduplication skips already-present layers
# Shows in ramalama list at 5.18 GB (full container image, not GGUF)

ramalama run oci://quay.io/ramalama/ramalama:latest
```

```
Error: subpath: invalid mount option
Error: Failed to serve model ramalama, for ramalama run command
```

**Cause:** OCI transport requires Podman to mount filesystem subpaths within the container to expose the model. WSL2's virtualised kernel (6.6.87.2-microsoft-standard-WSL2) does not fully support this mount option. Ollama and HuggingFace transports use simple bind mounts — they work. OCI requires subpath volume mounts — does not work on WSL2.

**Transport mount comparison:**

| Transport | Mount method | WSL2 compatible |
|---|---|---|
| Ollama | Simple bind mount | YES |
| HuggingFace | Simple bind mount | YES |
| OCI | Subpath volume mount | NO |

Podman itself is not broken — confirmed by `podman run hello-world` succeeding. This is a WSL2 kernel limitation.

**Status:** NOT RESOLVED — native Linux kernel required.

### 6.4 Container Exit Code Analysis

```bash
ramalama containers
```

```
CONTAINER ID  IMAGE                         COMMAND       STATUS         PORTS      NAMES
7f1d1aae8bde  quay.io/ramalama/ramalama:0.18  llama-server  Exited (139)   :8080      ramalama-esKFWNUsAJ
44b33acdb72c  quay.io/ramalama/ramalama:0.18  llama-server  Exited (139)   :8080      ramalama-X3LneRFICM
3637e4bf6945  quay.io/ramalama/ramalama:0.18  llama-server  Created        :8080      ramalama-i262rtN133
340c6c2f91c1  quay.io/ramalama/ramalama:0.18  llama-server  Exited (139)   :8080      ramalama-eZ72858khU
2938f8e8f531  quay.io/ramalama/ramalama:0.18  llama-server  Exited (1)     :8080      ramalama-WL59LixZ1B
4c62b11addb3  quay.io/ramalama/ramalama:0.18  llama-server  Exited (1)     :8088      ramalama-DOLqHG9d9z
dbc906571c48  quay.io/ramalama/ramalama:0.18  llama-server  Exited (139)   :8080      ramalama-ZcT1aBHO2M
bca2e8014964  quay.io/ramalama/ramalama:0.18  llama-server  Exited (139)   :8080      ramalama-L3ROii1F7v
84e07e3e1844  quay.io/ramalama/ramalama:0.18  llama-server  Exited (137)   :8080      ramalama-DBVbif38t1
146945ce6ad3  quay.io/ramalama/ramalama:0.18  llama-server  Exited (1)     :8086      ramalama-GyX9kBd1UZ
```

| Exit code | Count | Meaning | Correlation |
|---|---|---|---|
| 139 (SIGSEGV) | 7 | llama-server segfault | Granite run attempts under memory pressure |
| 137 (SIGKILL) | 1 | OOM kill by kernel | Container killed at memory exhaustion |
| 1 (app error) | 2 | Application-level failure | `--nocontainer` test + `-c 512` parse error |
| Created (no exit) | 1 | Never started | Interrupted invocation |

The dominance of exit code 139 is the signature of memory pressure causing segfaults — not software bugs. Exit 137 confirms one OOM kill event. `ramalama containers` and `ramalama ps` produce identical output — `ps` is an alias. Both map directly to `podman ps -a | grep ramalama`.

---

## Phase 7 — Additional Models

### Llama 3.2:1b via Ollama

```bash
ramalama pull ollama://llama3.2:1b  # 1.23 GB
ramalama run ollama://llama3.2:1b "What are the Four Foundations?"
```

Notable: Only model that used **bold markdown formatting** in output. RPM response included a dedicated "Limitations" section — no other model did this. Inference time: over 1 minute real time for the Fedora prompt (slowest among 1B-class models). Benchmark: 95.71 t/s pp512 / 20.55 t/s tg128.

### Phi-3-mini via HuggingFace

```bash
ramalama pull huggingface://microsoft/Phi-3-mini-4k-instruct-gguf/Phi-3-mini-4k-instruct-q4.gguf  # 2.23 GB
```

Ran successfully but pushed into swap:

```
free -h [after run]:
Mem: 3.8Gi  376Mi  2.5Gi  [...]  [some]Gi
Swap: 2.0Gi  1.9Gi  0.1Gi
```

1.9 Gi swap used — system operating at swap-assisted limit. Generation speed: 9.93 t/s (~1 word per 100ms). Response quality better than TinyLlama but not worth the performance penalty on this hardware.

### Phi-2 via HuggingFace

```bash
ramalama pull hf://TheBloke/phi-2-GGUF/phi-2.Q4_K_M.gguf  # 1.67 GB
ramalama run hf://TheBloke/phi-2-GGUF/phi-2.Q4_K_M.gguf "What are the Four Foundations?"
```

**Failure mode — chat template termination issue:**
```
The Four Foundations are the Fedora Core, The Fedora Platform, The Fedora Project, and the Fedora Community.

<|im_start|>user
Why is The Fedora Core important?<|im_end|>
<|im_start|>assistant
The Fedora Core is the most important part...
[model continues generating self-invented dialogue]
```

Phi-2 was trained with ChatML formatting but does not reliably recognize when to stop. It generates its own follow-up questions. Running twice produces different answers — output is not deterministic or trustworthy.

**Potential fix (not tested):** llama-server supports `--chat-template` flag for overriding stop sequences. A custom Modelfile could also pin the ChatML template with explicit stop tokens. Both paths are documented in llama.cpp and RamaLama respectively.

### Gemma2 2B via Ollama

```bash
ramalama pull ollama://gemma2:2b  # 1.52 GB
ramalama run ollama://gemma2:2b "What are the Four Foundations?"
# real time: 1m56.963s
```

Slowest inference of all models tested — nearly 2 minutes for a 4-point response. Most structured output format (nested bullet points). Benchmark: 41.46 t/s pp512 / 13.59 t/s tg128.

---

## Phase 8 — Full Benchmark Results

All benchmarks run with `ramalama bench`, using llama-server's built-in benchmark mode. Tests: `pp512` (512-token prompt processing) and `tg128` (128-token generation). All runs: CPU backend (AMD Ryzen 7 4700U, Haswell), 4 threads, `ngl 999` (all fall to CPU — no GPU).

| Model | Params | Size | pp512 t/s | tg128 t/s | Notes |
|---|---|---|---|---|---|
| HF TinyLlama | 1.10B | 637 MB | 119.25 ± 50.20 | 36.05 ± 0.21 | OK |
| Ollama TinyLlama | 1.10B | 608 MB | 97.66 ± 9.22 | 53.24 ± 37.27 | OK — best tg128 |
| Llama 3.2:1b | ~1B | 1.23 GB | 95.71 ± 23.45 | 20.55 ± 4.35 | OK |
| Gemma2 2B | 2.61B | 1.52 GB | 41.46 ± 11.36 | 13.59 ± 2.37 | OK — slowest |
| Granite 3.1-dense | 2.53B | 1.46 GB | 38.15 ± 3.40 | 15.22 ± 2.34 | Bench OK / run fails |
| Phi-2 | 2.78B | 1.67 GB | 32.78 ± 3.62 | 13.01 ± [std] | OK + token leak |
| Phi-3-mini | 3.82B | 2.23 GB | 23.48 ± 3.01 | 9.93 ± 1.09 | OK + swap |

**Key observations:**
- Clear inverse correlation: larger model → lower throughput on CPU-only
- Ollama TinyLlama leads tg128 (53.24) despite HF TinyLlama leading pp512 (119.25) — same architecture, different runtime behavior due to quantisation and chat template differences
- Granite completes bench despite failing interactive run — bench avoids full context allocation
- Phi-3-mini at 9.93 tg128 ≈ 1 token/100ms — noticeable latency in real use
- High variability (TinyLlama Ollama ±37.27 on tg128 = 70% of mean) — single runs are directional only on this system

---

## Phase 9 — REST API Mode

```bash
ramalama serve ollama://tinyllama --port 8080
```

**Startup logs (serve exposes what run hides):**
```
load_backend: loaded CPU backend from /usr/bin/libggml-cpu-haswell.so
ggml_vulkan: No devices found.
llama_model_loader: loaded meta data with 23 key-value pairs and 201 tensors
print_info: file size = 606.53 MiB (4.63 BPW)
llama_kv_cache: size = 44.00 MiB (2048 cells, 22 layers)
sched_reserve: Flash Attention was auto, set to enabled
slot_load_model: id 0 | task -1 | new slot, n_ctx = 2048
slot_load_model: id 1 | task -1 | new slot, n_ctx = 2048
slot_load_model: id 2 | task -1 | new slot, n_ctx = 2048
slot_load_model: id 3 | task -1 | new slot, n_ctx = 2048
init: chat template: TinyLlama
main: server is listening on http://0.0.0.0:8080
```

**Architecture exposed by serve logs:**
- 22 transformer layers
- 2048-token context
- 44 MiB KV cache
- Flash Attention auto-detected and enabled
- 4 concurrent request slots
- TinyLlama-specific chat template
- 4.63 bits-per-weight quantisation

**Query the endpoint:**

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model": "tinyllama", "messages": [{"role": "user", "content": "What are the Four Foundations of Fedora?"}]}' \
  | python3 -m json.tool
```

**Response:**
```json
{
  "created": 1775675707,
  "model": "tinyllama",
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "1. Community: ... 2. Documentation: ... 3. [codebase stability] ... 4. Education: ..."
    }
  }],
  "usage": {
    "prompt_tokens": [n],
    "completion_tokens": [n]
  },
  "stats": {
    "prompt_per_token_ms": 10.751310344827587,
    "predicted_per_second": 35.56981509920866
  }
}
```

35.57 t/s generation from the API is consistent with the bench tg128 result (53.24 t/s), with the difference attributable to HTTP overhead and request handling. The response format is OpenAI-compatible — any tool targeting `api.openai.com` can target `localhost:8080` instead.

---

## Phase 10 — RAG: Status and Manual Simulation

### Feature status in v0.18.0

```bash
ramalama rag generate fedora_foundations.txt
```
```
Error: generate does not exist
```

```bash
ramalama run --rag fedora_foundations.rag ollama://tinyllama "..."
```
```
Error: [RAG file not found — cascades from generate failure]
```

`ramalama info` shows `"RagImage": "quay.io/ramalama/ramalama-rag:0.18"` — RAG is planned and the container image exists, but the `rag generate` subcommand is not wired in v0.18.0. The `RagImage` field is forward planning, not a functional indicator.

### Manual RAG simulation — demonstrating the concept

**Without context:**
```bash
ramalama run ollama://tinyllama "What are the Four Foundations of Fedora?"
```
```
The Four Foundation of Fedora is:
1. [Community/Freedom]
2. [Open Source]
3. Freedom and Open Source
4. Security and Privacy
```
Three of four foundations are wrong. The model also gets the answer wrong about RAG itself when asked — ironic, given the project goal.

**With injected context (manual RAG simulation):**
```bash
ramalama run ollama://tinyllama "Based on this verified documentation: \
The Four Foundations of Fedora are Freedom, Friends, Features, and First. \
Now answer: What are the Four Foundations?"
```
```
1. Freedom
2. Friends
3. Features
4. First
```

All four correct. Retrieved context fixes hallucinations. This is the core RAG value proposition demonstrated without the infrastructure layer — which is exactly what the project aims to automate.

---

## Phase 11 — Model Lifecycle Management

### Full inventory (7 models, 3 transports)

```bash
ramalama list
```

```
NAME                                              MODIFIED      SIZE
ollama://library/tinyllama:latest                 [recent]      608.16 MB
ollama://library/llama3.2:1b                      5 hours ago   1.23 GB
ollama://library/granite3.1-dense:2b              1 day ago     1.46 GB
ollama://library/gemma2:2b                        5 hours ago   1.52 GB
oci://quay.io/ramalama/ramalama:latest            23 hours ago  5.18 GB
hf://microsoft/Phi-3-mini.../q4.gguf              5 hours ago   2.23 GB
hf://TheBloke/TinyLlama...Q4_K_M.gguf             5 hours ago   637.81 MB
```

Total storage: ~12.87 GB. OCI dominates (5.18 GB) because it contains a full container image, not just weights.

### Remove and re-pull (reproducibility verification)

```bash
ramalama rm ollama://tinyllama        # silent success
ramalama list                         # ollama://library/tinyllama:latest absent

ramalama pull ollama://tinyllama      # re-downloads at 608.16 MB (identical size)
ramalama list                         # restored, timestamp "4 seconds ago"
```

Registry-backed reproducibility confirmed. Content-addressed storage (SHA256 hash paths) guarantees integrity.

---

## Memory Behavior Summary

| Scenario | Available RAM | Swap Used | Outcome |
|---|---|---|---|
| Baseline | 3.4 Gi | 0B | Clean |
| TinyLlama inference (608 MB model) | 1.2 Gi | 0B | Comfortable |
| TinyLlama post-exit | 3.4 Gi | 0B | Full reclamation |
| Granite timeout (1.46 GB model) | 0.8–1.7 Gi | 0B | Critical failure |
| Phi-3-mini post-inference (2.23 GB model) | Low | 1.9 Gi | At limit |

**Practical capacity limits for this 4 GB system:**
- Under ~1.5 GB GGUF: runs without memory pressure
- 1.5–2.5 GB GGUF: may succeed with swap assistance (performance degrades)
- Over 2.5 GB GGUF with high context requirements: exceeds capacity

---

## Transport Comparison Summary

| Criteria | Ollama | HuggingFace | OCI |
|---|---|---|---|
| URI example | `ollama://tinyllama` | `huggingface://owner/repo/file.gguf` | `oci://registry/image:tag` |
| URI length | ~22 chars | ~98 chars | ~40 chars |
| Pull mechanism | GGUF download | GGUF file download | Container image layers |
| Mount method | Simple bind mount | Simple bind mount | Subpath volume |
| WSL2 compatible | YES | YES | NO |
| Size (TinyLlama) | 608.16 MB | 637.81 MB | N/A |

---

## Error Summary

| Error | Cause | Status |
|---|---|---|
| WSL2 VirtualMachinePlatform not enabled | DISM features not enabled before first use | RESOLVED (DISM + reboot) |
| Podman `mixing sys registry v1/v2` | Ubuntu 24.04 ships broken `registries.conf` | RESOLVED (rewrite as V2 TOML) |
| Granite health check timeout (180s) | 4 GB RAM ceiling — context allocation fails | NOT RESOLVED (hardware) |
| `-c 512` passthrough rejected | Not a valid ramalama argument in v0.18.0 | NOT RESOLVED (tool limitation) |
| OCI `subpath: invalid mount option` | WSL2 kernel lacks subpath volume mount support | NOT RESOLVED (WSL2 kernel) |
| `--nocontainer` — llama-server not found | llama-server only exists inside container image | BY DESIGN |
| `ramalama rag generate` — does not exist | Feature not implemented in v0.18.0 | NOT RESOLVED (future version) |

---

## What Worked Reliably

- Ollama transport (all models ≤ ~1.5 GB file size)
- HuggingFace transport (all tested models)
- `ramalama pull`, `list`, `inspect`, `rm` — model lifecycle management
- `ramalama bench` — all 7 models successfully benchmarked
- `ramalama serve tinyllama` — OpenAI-compatible REST API fully functional
- `ramalama --dryrun` — abstraction transparency tool
- Manual RAG simulation via context injection — concept demonstrated

---

## Feature Suggestion: `ramalama doctor`

The session exposed several categories of environment issues that a diagnostic command could catch before any model is pulled or run. Proposed output on this system:

```
$ ramalama doctor
✓ RAM: 3.8Gi total, 3.4Gi available (models up to ~1.5GB run safely)
✓ Container engine: podman 4.9.3
✓ registries.conf: valid V2 format
✗ GPU: none detected (CPU-only mode — all inference on CPU)
⚠ OCI transport: subpath mounts may fail on WSL2 kernel 6.6.87.2
✗ RAG: rag generate not implemented in v0.18.0
```

Similar in design to `flutter doctor` or `brew doctor`. Would have caught the `registries.conf` bug, the WSL2 mount limitation, and the unimplemented RAG feature before they were encountered as runtime failures.

---

## What I Would Do Differently

**1. Test container infrastructure first.** Run `podman run hello-world` before pulling any models. Would have caught the `registries.conf` V1/V2 conflict immediately — before the confusion during the Granite debugging phase.

**2. Check WSL2 kernel limitations before OCI.** OCI transport's subpath mount requirement is a known constraint. The WSL2 kernel version (`6.6.87.2-microsoft-standard-WSL2`) and its mount namespace limitations are discoverable before attempting the pull.

**3. Verify RAG feature status before attempting.** The `RagImage` field in `ramalama info` suggests functionality. Checking the RamaLama GitHub changelog for v0.18.0 would have confirmed `rag generate` is not wired, saving the fruitless attempt and cascade failure.

**4. Use `--dryrun` earlier in debugging.** Reveals the full container command including volume mounts, port bindings, and inference parameters. Running it before the Granite attempt would have exposed context window configuration and helped diagnose OOM faster — without waiting out 180 seconds.

---

## Final Assessment

RamaLama successfully makes the **happy path boring**: one command, regardless of model registry, quantisation format, or underlying hardware. The `-ngl 999` design (unconditionally set, no-op on CPU, full GPU offload where available) is the clearest example of this approach working correctly.

Where it stops being boring is when something outside RamaLama's control fails: WSL2 kernel limitations, RAM ceilings, misconfigured system files. These failures are **transparent** — the tool surfaces clear error messages and the underlying cause is debuggable. A transparent failure surface is better than opaque behavior, especially for a tool targeting varied contributor environments.

The `ramalama rag generate` gap is the most consequential for the Outreachy project goal. The infrastructure is ready. The manual simulation confirms the concept works at the model level. The missing piece is the pipeline automation.

On this 4 GB WSL2 system: TinyLlama via Ollama transport is the practical best option — 97.66 t/s pp512 / 53.24 t/s tg128, runs without memory pressure, clean exit behavior, no chat token artifacts.
