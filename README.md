# RamaLama Exploration Report

**Contributor:** Akriti Sengar | **Outreachy 2026 · Fedora Project**

## System Setup

| Component | Specification |
|-----------|---------------|
| Host | Windows 11, AMD Ryzen 7 4700U, 7.4 GB RAM |
| WSL2 | Ubuntu 24.04, kernel 6.6.87.2, 4 GB RAM cap |
| Tool | RamaLama 0.18.0, Podman 4.9.3, CPU-only |

## Task Completion

### 1. Install RamaLama
```bash
pip install ramalama --break-system-packages
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
ramalama version'
```
Output: `ramalama version 0.18.0`

2. First Model — TinyLlama (Ollama transport)
```bash
ramalama pull ollama://tinyllama # Downloaded: 608.16 MB at 12.30 MB/s

ramalama run ollama://tinyllama "What are the Four Foundations of the Fedora project?"
```
Output:
```
1. Community: Fedora prioritizes community participation
2. Innovation: Fedora values new technologies
3. Freedom: Software must be free
4. Continuous Integration: Fedora uses CI for quality
```

> Analysis: TinyLlama got Freedom right, Community loosely maps to Friends, but Innovation and Continuous Integration are hallucinations. The model ran smoothly using ~2.2 GB RAM on my 4 GB WSL2 cap.

3. Second Model — Different Transport (HuggingFace)
```bash
ramalama pull huggingface://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf # Downloaded: 637.81 MB (29 MB larger than Ollama version)

ramalama run huggingface://.../tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf "What are the Four Foundations?"
```

Output:
```
1. Community: cornerstone of the project
2. Dedication: deep level of dedication and support
3. Community Values: values community as core
4. Community Success: success through community values
```

> Comparison: Same model weights (1.1B, 201 tensors), completely different output. Ollama's version hallucinated technical concepts; HuggingFace's version looped on "community" four times. The difference is packaging — different system prompts and chat templates between registries.

4. Third Model — Larger Model (Granite 3.1-dense:2b)
```bash
ramalama pull ollama://granite3.1-dense:2b # Downloaded: 1.46 GB

ramalama run ollama://granite3.1-dense:2b "What are the Four Foundations?"
```
Output:
```
............................................................................
[180 seconds of dots]
ERROR - Command 'health check of container' timed out after 180 seconds
```
> Failure Analysis: Granite (2.53B parameters) loaded its 1.46 GB weights successfully but failed at context allocation. My 4 GB WSL2 cap left ~3.4 GB usable RAM; Granite needed more. The ramalama serve logs revealed: common_init from params: failed to create context with model. This is a hardware constraint, not a tool bug.

5. What Worked (and What Didn't)

| Transport |	Model |	Status | Notes |
|-----------|-------|--------|-------|
| Ollama	| TinyLlama (608 MB) |	✅ |	53 t/s generation, clean exit |
| HuggingFace |	TinyLlama (637 MB) |	✅	| 119 t/s prompt processing |
| Ollama	| Granite (1.46 GB)	| ❌	| OOM on context allocation |
| OCI |	ramalama:latest |	❌	| WSL2 subpath mount incompatibility |

**Also discovered/fixed:** Podman's `registries.conf` had V1/V2 mixing bug — rewrote as clean V2 TOML, resolved.

Does RamaLama Make AI Boring?
Yes, and that's a compliment.

The `--dryrun` flag reveals what's happening under the hood:

```bash
ramalama --dryrun run ollama://tinyllama "prompt" # Generates an 18+ flag podman command with security opts, GPU config, port mappings, etc.
```
A single `ramalama run` command abstracts away:

Container security flags (--cap-drop=all, --security-opt=no-new-privileges)

GPU layer configuration (-ngl 999 — works on CPU as no-op, full GPU if available)

Model path resolution across 3 different registry types

Port binding and container lifecycle

The boring part: When everything works, I type one command and get an answer. I don't think about containers, volume mounts, or model paths. The complexity is handled so competently that I never have to touch it.

The not-boring-yet part: Hardware constraints (4 GB RAM killed Granite), WSL2 kernel limitations (OCI transport fails), and missing features (RAG not implemented in v0.18.0) still require expert debugging. The happy path is boring. The edge cases still need work — which is exactly what this Outreachy project aims to solve.

Key Takeaways
Same model, different transport = different behavior. Ollama and HuggingFace TinyLlama gave completely different answers to the same prompt.

RamaLama abstracts complexity well — but can't paper over hardware limits. Granite failed predictably on my 4 GB system.

The RAG value proposition is real. Manual context injection fixed TinyLlama's hallucinations — that's what we'll automate.

bash
# Without context — wrong
ramalama run ollama://tinyllama "What are the Four Foundations?"
# Output: Community, Innovation, Freedom, CI (3 wrong)

# With injected context — correct
ramalama run ollama://tinyllama "Based on docs: Freedom, Friends, Features, First. Now answer?"
# Output: Freedom, Friends, Features, First (all correct)
text

---

**What I changed:**

| Original (800+ lines) | Shortened (150 lines) |
|----------------------|----------------------|
| Full benchmark tables | Removed — keep in full report |
| All 7 model details | Focus on 3 representative models |
| Container exit code analysis | Removed — interesting but not required |
| Memory tracking tables | Brief mention only |
| Step-by-step WSL2 setup | Condensed to system table |
| RAG deep dive | Single demonstration block |

**What I kept:**
- All 5 required steps with commands + outputs
- Comparison across transports (Ollama vs HuggingFace)
- Failure analysis with root cause
- Your opinion on "boring"
- The RAG manual demo (core to the project)

Your `My.docx` is the **full engineering report**. The `README.md` should be the **executive summary** that someone reads before deciding if they want the full report. This shortened version does that.
