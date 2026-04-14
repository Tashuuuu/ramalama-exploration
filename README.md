# ramalama-lab — run local LLMs without the headache

A systematic exploration of **RamaLama 0.18.0**: pull, run, benchmark, and serve AI models locally using a single command.

Tested on: **Windows 11 + WSL2 Ubuntu 24.04, 4 GB RAM cap, AMD Ryzen 7 4700U, no GPU.**  
Low-end hardware, CPU-only, no GPU, inside a VM. Documented in detail so you can calibrate what will and won't work on your own setup.

---

## Quick start (if you already know RamaLama)

```bash
pip install ramalama --break-system-packages
ramalama pull ollama://tinyllama
ramalama run ollama://tinyllama "What are the Four Foundations of Fedora?"
```

> **Ubuntu 24.04 users:** Fix a Podman registry config bug before any of this — step 1 has the one-command fix. Skip it and every model pull will fail with a cryptic registry error.

Everything else below is the full walkthrough — setup, the five required tasks with complete terminal output, benchmarks, and why things break.

---

## What you need

- Windows 11 with WSL2 (or native Linux)
- Ubuntu 24.04 LTS
- Python 3.12+
- Podman 4.9.3+
- ~4 GB RAM (more is better; less will hurt)

**WSL2 `.wslconfig` used in this setup:**
```ini
[wsl2]
memory=4GB
processors=4
swap=2GB
```

![.wslconfig output]()

> The 4 GB cap gives 3.8 Gi total inside WSL2 after ~200 MB of VM/hypervisor overhead, and then ~3.4 Gi actually available at baseline after WSL2's own processes take their share. That 3.4 Gi number is the real headroom for inference.

---

## WSL2 Setup (Windows only)

Skip this section if you're on native Linux.

```powershell
# Enable required Windows features
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Reboot — required for features to activate
shutdown /r /t 0
```

After reboot:

```powershell
wsl --install -d Ubuntu-24.04
wsl --set-default Ubuntu-24.04
```

Verify:
```powershell
wsl --list --verbose
```
```
NAME          STATE   VERSION
Ubuntu-24.04  Running 2
```

---

## 1. Install dependencies inside WSL2

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv podman
```

Verify Podman:
```bash
podman --version
```
```
podman version 4.9.3
```

**Fix a Ubuntu 24.04 Podman bug before going further.**

Ubuntu 24.04's default Podman ships a `/etc/containers/registries.conf` that mixes V1 and V2 format entries — Podman refuses to parse it and every `pull` fails with a registry configuration error. Fix it now, before you touch any models:

```bash
sudo cp /etc/containers/registries.conf /etc/containers/registries.conf.bak

sudo tee /etc/containers/registries.conf << 'EOF'
unqualified-search-registries = ["docker.io", "quay.io"]
EOF
```

Verify the fix works:
```bash
podman run --rm docker.io/library/hello-world
```
```
Hello from Docker!

This message shows that your installation appears to be working correctly.
...
```

If you see that, the container stack is healthy. If you skip this step and go straight to pulling models, you'll hit a confusing registry parsing error that has nothing to do with RamaLama.

---

## 2. Install RamaLama  *(Task 1)*

```bash
pip install ramalama --break-system-packages
```
```
Defaulting to user installation because normal site-packages is not writeable
Collecting ramalama
  Downloading ramalama-0.18.0-py3-none-any.whl (204 kB)
Collecting argcomplete (from ramalama)
  Downloading argcomplete-3.6.3-py3-none-any.whl (43 kB)
Requirement already satisfied: pyyaml in /usr/lib/python3/dist-packages
Requirement already satisfied: jsonschema in /usr/lib/python3/dist-packages
Requirement already satisfied: jinja2 in /usr/lib/python3/dist-packages
WARNING: The script ramalama is installed in '~/.local/bin' which is not on PATH.
Successfully installed argcomplete-3.6.3 ramalama-0.18.0
```

The two yellow WARNING lines are the only anomaly — the install itself succeeds cleanly. The PATH issue is a known consequence of user-level pip installation on Ubuntu 24.04. Fix it:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

> Ubuntu 24.04 blocks pip by default (PEP 668). `--break-system-packages` is fine for exploration. For anything more permanent: `python3 -m venv venv && source venv/bin/activate`.

---

## 3. Verify the install  *(Task 2)*

```bash
ramalama version
```
```
ramalama version 0.18.0
```

Run `ramalama info` for the full system picture — it's more useful than it looks:

```bash
ramalama info
```
```json
{
  "Accelerator": "none",
  "Engine": {
    "Info": {
      "host": {
        "arch": "amd64",
        "buildahVersion": "1.33.7",
        "cgroupVersion": "v2",
        "kernel": "6.6.87.2-microsoft-standard-WSL2",
        "networkBackend": "netavark",
        "ociRuntime": { "path": "/usr/bin/crun" },
        "swapTotal": 2147483648
      }
    },
    "Version": { "Version": "4.9.3" }
  },
  "RagImage": "quay.io/ramalama/ramalama-rag:0.18",
  "UseContainer": true,
  ...
}
```

Two things to note right away: `"Accelerator": "none"` means everything runs on CPU — confirmed before a single model is pulled. And `"RagImage"` shows RAG is planned but the field's presence doesn't mean it works yet (more on that in the failures section).

---

## 4. Pull and run your first model — Ollama transport  *(Tasks 3 and 4)*

Before pulling anything, run the dryrun. It doesn't execute — it just prints the full Podman command RamaLama would construct internally:

```bash
ramalama --dryrun run ollama://tinyllama "What are the Four Foundations of the Fedora project?"
```
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

That's 18+ flags assembled from a single short command. Security hardening, GPU layer config (`-ngl 999` — a no-op here but enables full acceleration on machines that have a GPU), port mapping, inference parameters. You don't need to understand all of it. That's exactly the point.

### Pull

```bash
ramalama pull ollama://tinyllama
```
```
Downloading ollama://library/tinyllama:latest ...
Trying to pull ollama://library/tinyllama:latest ...
Downloading tinyllama
100% |████████████████████| 608.16 MB/608.16 MB 12.30 MB/s 0s
```

`ollama://tinyllama` resolves to `ollama://library/tinyllama:latest` automatically. The model file is stored at `~/.local/share/ramalama/store/ollama/library/tinyllama/`.

### Confirm it's there

```bash
ramalama list
```
```
NAME                                    MODIFIED        SIZE
ollama://library/tinyllama:latest       32 seconds ago  608.16 MB
```

### Inspect the metadata

```bash
ramalama inspect ollama://tinyllama
```
```
tinyllama
Path:     ~/.local/share/ramalama/store/ollama/library/tinyllama/blobs/sha256-2af3b81862c6b...
Registry: ollama
Version:  3
Endian:   little
[201 tensors — GGUF V3 format]
```

GGUF V3, little-endian, 201 tensors. This is the baseline you'll compare against when the HuggingFace pull comes back with the same model but different metadata.

### Run

```bash
ramalama run ollama://tinyllama "What are the Four Foundations of the Fedora project?"
```
```
1. Community: Fedora prioritizes community participation as its core value.
2. Innovation: Fedora values incorporating new technologies and advancements.
3. Freedom: Software must be free to use, modify, and distribute.
4. Continuous Integration: Fedora uses a continuous integration system to ensure
   the quality and consistency of its code and documentation.
```

The actual Four Foundations are **Freedom, Friends, Features, First**. TinyLlama got Freedom correct, mapped Community loosely to Friends, then hallucinated Innovation and Continuous Integration with the same confident tone as the correct ones. That's a 1.1B parameter model doing what 1.1B parameter models do. The hallucination problem is exactly what RAG exists to fix.

---

## 5. A different transport — HuggingFace

Same model family, different registry. HuggingFace TinyLlama was chosen deliberately — keeping the architecture constant isolates transport as the variable. Any difference in output, size, or benchmark is down to how each registry packages the model, not the model itself. This is where transport differences stop being abstract.

### Pull

```bash
ramalama pull huggingface://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf
```
```
Downloading hf://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf ...
Trying to pull hf://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf ...
Downloading tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf
100% |████████████████████| 637.81 MB/637.81 MB 12.89 MB/s 0s
```

98 characters in the identifier versus 22 for Ollama. Same underlying model, but the HuggingFace path makes you name the repository owner, the repo, and the specific quantisation file explicitly. The file came in 29.65 MB heavier than the Ollama version — different quantisation metadata and chat template packaging.

### Both models now in the store

```bash
ramalama list
```
```
NAME                                                                                                    MODIFIED        SIZE
ollama://library/tinyllama:latest                                                                       13 minutes ago  608.16 MB
hf://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf                    32 seconds ago  637.81 MB
```

### Inspect the metadata

```bash
ramalama inspect huggingface://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf
```
```
tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf
Path:     ~/.local/share/ramalama/store/huggingface/TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/...
Registry: huggingface
Version:  3
Endian:   little
[201 tensors — GGUF V3 format]
```

Both report `Registry`, `Version: 3`, `Endian: little`, and 201 tensors — the architecture is identical. The only inspect-level difference is the registry field and the storage path under `~/.local/share/ramalama/store/`. What changes at runtime is the chat template each registry bundles with the GGUF file.

### Run — same prompt, same model, different result

```bash
ramalama run huggingface://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf \
  "What are the Four Foundations of the Fedora project?"
```
```
The Four Foundation of the Fedora project are:
1. Community: Fedora is a community project, and the community is the cornerstone.
   Fedora users are the foundation of the project and the most important contributors.
2. Dedication: Fedora is a community project, and the community has a deep level of
   dedication and support.
3. Community Values: The project values community, and community values are the core.
4. Community Success: Fedora has achieved success through community values.
```

Same weights, same architecture, same prompt. Completely different answer. The Ollama version hallucinated technical concepts (Innovation, Continuous Integration). The HuggingFace version got stuck in a loop on "Community" — four differently-worded community-themed points, none of them the actual foundations. Both confidently wrong, but wrong in different directions. The chat template and system prompt packaging is the likely cause, not the model weights.

---

## 6. Transport comparison: all three

| Transport | Pull command | Size | WSL2 |
|-----------|-------------|------|-------|
| Ollama | `ramalama pull ollama://tinyllama` | 608.16 MB | ✅ |
| HuggingFace | `ramalama pull huggingface://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf` | 637.81 MB | ✅ |
| OCI | `ramalama pull oci://quay.io/ramalama/ramalama:latest` | 5.18 GB¹ | ❌ run fails² |

¹ `quay.io/ramalama/ramalama:latest` is the RamaLama runtime image — it includes llama.cpp and all dependencies, not just weights, which explains the size. The WSL2 subpath mount failure is a kernel limitation, not specific to this image.  
² OCI uses subpath volume mounts that the WSL2 kernel doesn't support — runs fine on native Linux.

**Why OCI fails on WSL2:** OCI transport uses subpath volume mounts. The WSL2 kernel (`6.6.87.2-microsoft-standard-WSL2`) doesn't support them.

```
Error: subpath: invalid mount option
```

On native Linux, OCI transport works fine.

---

## 7. Benchmark results (CPU-only, 4 threads)

All runs: AMD Ryzen 7 4700U, no GPU, llama.cpp backend.

```bash
ramalama bench ollama://tinyllama
```

| Model | Params | Size | pp512 (t/s) | tg128 (t/s) | Interactive run |
|-------|--------|------|------------|------------|-----------------|
| HF TinyLlama | 1.10B | 638 MB | 119.25 | 36.05 | ✅ |
| Ollama TinyLlama | 1.10B | 608 MB | 97.66 | **53.24** | ✅ |
| Llama 3.2 1B | ~1B | 1.23 GB | 95.71 | 20.55 | ✅ (slow) |
| Gemma2 2B | 2.61B | 1.52 GB | 41.46 | 13.59 | ✅ (~2 min/query) |
| Granite 3.1-dense | 2.53B | 1.46 GB | 38.15 | 15.22 | ❌ OOM |
| Phi-2 | 2.78B | 1.67 GB | 32.78 | 13.01 | ✅ + chat token leak |
| Phi-3-mini | 3.82B | 2.23 GB | 23.48 | 9.93 | ✅ (slow) |

**Rule of thumb on a 4 GB system:**
- Under ~1.5 GB: runs comfortably, no swap
- 1.5–2.5 GB: runs with moderate overhead — minimal swap (74 Mi for Phi-3-mini), but noticeably slower
- Above 2.5 GB + large context: fails at context allocation

> **Note on TinyLlama variability:** tg128 shows ±37.27 t/s standard deviation — 70% of the mean. Likely causes: CPU thermal throttling under sustained inference load and background process competition on WSL2. Treat as directional, not precise.

---

## 8. REST API mode

```bash
# Terminal 1 — start the server
ramalama serve ollama://tinyllama --port 8080
```

Server confirms:
```
main: server is listening on http://0.0.0.0:8080
```

```bash
# Terminal 2 — query it
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"tinyllama","messages":[{"role":"user","content":"What is Fedora?"}]}' \
  | python3 -m json.tool
```

The endpoint is OpenAI-compatible — anything that works with the OpenAI API (LangChain, llama_index, etc.) can point here with just a base URL swap.

Observed throughput via REST: **35.57 tokens/second** generation (consistent with bench results; the small gap is HTTP overhead).

---

## 9. Where things break (and why)

### ❌ Granite 3.1-dense:2b — OOM on 4 GB

```bash
ramalama pull ollama://granite3.1-dense:2b   # ✅ 1.46 GB — downloads fine
ramalama run ollama://granite3.1-dense:2b    # ❌ times out after 180 seconds
```

```
Error: Command 'health check of container ramalama-eZ72858khU' timed out after 180 seconds
```

**Why:** Granite loads its 1.46 GB weights successfully, but context window allocation fails. The `llama-server` log shows `common_init from params: failed to create context with model`. At 32 attention heads with 8 KV heads and default context length, the KV cache needs more RAM than the system has available.

**Interesting:** `ramalama bench ollama://granite3.1-dense:2b` *succeeds* — bench uses short bursts that avoid full context allocation.

**Fix:** Use a smaller model (TinyLlama) or increase WSL2 RAM cap.

### ❌ OCI transport on WSL2

```bash
ramalama pull oci://quay.io/ramalama/ramalama:latest   # ✅ pulls fine
ramalama run oci://quay.io/ramalama/ramalama:latest    # ❌ mount error
```

```
Error: subpath: invalid mount option
```

**Why:** WSL2's virtualised kernel doesn't support the subpath volume mounts that OCI transport requires. Ollama and HuggingFace use simple bind mounts — those work fine.

**Fix:** Run on native Linux.

### ❌ RAG feature — not yet in v0.18.0

```bash
ramalama rag generate mydoc.txt
```
```
Error: generate does not exist
```

`ramalama info` shows a `RagImage` field pointing to `quay.io/ramalama/ramalama-rag:0.18` — it's planned, but the CLI isn't wired yet. That field in `ramalama info` is easy to mistake for confirmation that RAG works. It doesn't.

**Manual RAG simulation works** — inject context directly in the prompt:

```bash
# Without context — hallucinated answer
ramalama run ollama://tinyllama "What are the Four Foundations of Fedora?"
# → "Freedom and Open Source", "Security and Privacy" — both wrong

# With context injected — grounded answer
ramalama run ollama://tinyllama "Based on this documentation: The Four Foundations \
of Fedora are Freedom, Friends, Features, and First. Now answer: What are the Four \
Foundations of Fedora?"
# → All four named correctly (with minor misspelling: "Freeoms")
```

Same model, same hardware, same question. The only difference is the retrieved context. That gap between the two outputs is the entire argument for why RAG matters — and what the Outreachy project aims to automate.

### ❌ --nocontainer flag

```bash
ramalama --nocontainer run ollama://tinyllama "test"
```
```
Error: [Errno 2] No such file or directory: 'llama-server'
```

`llama-server` exists only inside the RamaLama container image. The containerized approach is not optional convenience — it's the delivery mechanism for the runtime itself.

---

## 10. Commands you'll actually use

| Command | What it does |
|---------|-------------|
| `ramalama version` | Confirm installation |
| `ramalama info` | Full system report (accelerator, engine, model aliases) |
| `ramalama pull ollama://tinyllama` | Download a model |
| `ramalama list` | Show all downloaded models |
| `ramalama inspect ollama://tinyllama` | Show model metadata (tensors, quantization, storage path) |
| `ramalama run ollama://tinyllama "prompt"` | Run a one-shot query |
| `ramalama serve ollama://tinyllama --port 8080` | Start OpenAI-compatible REST API |
| `ramalama bench ollama://tinyllama` | Throughput benchmark (pp512 + tg128) |
| `ramalama containers` | List all containers (running and exited) |
| `ramalama rm ollama://tinyllama` | Delete a model |
| `ramalama --dryrun run ollama://tinyllama "hi"` | Show the full podman command without executing |

---

## Memory reference

| Model | During inference | Swap used | Result |
|-------|-----------------|-----------|--------|
| TinyLlama (608 MB) | ~2.2 Gi consumed, 1.2 Gi available | 0B | ✅ Comfortable |
| Phi-3-mini (2.23 GB) | Runs with moderate overhead | 74 Mi | ✅ Completes successfully |
| Granite (1.46 GB + context) | ~0.8–1.7 Gi available | 0B | ❌ Context alloc fails |

Memory is fully reclaimed after container exit — no leaks between runs.

---

## Further reading

- **[BLOG.md](./BLOG.md)** — honest reflections on what "making AI boring" actually means in practice
