# little-coder

**A pi-based coding agent optimized for small local language models.**

little-coder started as a Claude Code-inspired Python CLI that adapts a cloud-style coding agent to 5–25 GB local models served via Ollama or llama.cpp. With v0.1.0 the agent has been ported onto **[pi](https://github.com/badlogic/pi-mono)** (`@mariozechner/pi-coding-agent`) as a set of TypeScript extensions. Every mechanism the whitepaper cites as load-bearing — Write-vs-Edit invariant, per-turn skill injection, algorithm-cheat-sheet injection, thinking-budget cap, output-parser, quality monitor, per-model profiles, evidence-aware compaction — is preserved as a pi extension.

**Paper result (v0.0.2):** `ollama/qwen3.5` (9.7B, 6.6 GB) + little-coder scored **45.56 % mean across two full runs** of the Aider Polyglot benchmark — above gpt-4.5-preview (44.9 %) and gpt-oss-120b high (41.8 %) on the public leaderboard. Matched-model vanilla Aider baseline: 19.11 %. The whitepaper — *Honey, I Shrunk the Coding Agent* — is published on Substack: **https://open.substack.com/pub/itayinbarr/p/honey-i-shrunk-the-coding-agent**. The exact codebase that produced those numbers is preserved at tag **[`v0.0.2`](https://github.com/itayinbarr/little-coder/releases/tag/v0.0.2)** (commit `1d62bde`) — check it out to reproduce.

**Best result (v0.0.5):** `llamacpp/qwen3.6-35b-a3b` (Qwen3.6-35B-A3B MoE, 22 GB Q4_K_M) + little-coder scored **78.67 %** on the same 225-exercise benchmark, running on an 8 GB laptop GPU. Tag **[`v0.0.5`](https://github.com/itayinbarr/little-coder/releases/tag/v0.0.5)** preserves that codebase; the full write-up is in [`docs/benchmark-qwen3.6-35b-a3b.md`](docs/benchmark-qwen3.6-35b-a3b.md).

**v0.1.0** is a heavy architectural upgrade that ports the agent onto pi without regressing the whitepaper's result path. See [`CHANGELOG.md`](CHANGELOG.md) for details.

---

## Quick start

### 1. Install Node.js 20+ and the little-coder dependencies

```bash
git clone https://github.com/itayinbarr/little-coder.git
cd little-coder
npm install
```

### 2. Serve a model locally

**Option A — llama.cpp** (fastest, supports MoE models like Qwen3.6-35B-A3B):

```bash
# Build llama.cpp with CUDA (sm_XXX matches your GPU; Blackwell = 120)
git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=120 -DLLAMA_CURL=ON
cmake --build build --config Release -j

# Fetch a GGUF (Qwen3.6-35B-A3B Q4_K_M, 22 GB)
pip install -U "huggingface_hub[cli]"
hf download unsloth/Qwen3.6-35B-A3B-GGUF Qwen3.6-35B-A3B-UD-Q4_K_M.gguf \
   --local-dir ~/models

# Serve (MoE trick: experts in RAM, attention on GPU)
build/bin/llama-server -m ~/models/Qwen3.6-35B-A3B-UD-Q4_K_M.gguf \
   --host 127.0.0.1 --port 8888 --jinja \
   -c 16384 -ngl 99 --n-cpu-moe 999 --flash-attn on
```

**Option B — Ollama** (simplest):

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen3.5
```

### 3. Run little-coder

```bash
cd /path/to/little-coder

# Interactive
LLAMACPP_API_KEY=noop ./node_modules/.bin/pi --model llamacpp/qwen3.6-35b-a3b

# Single prompt
LLAMACPP_API_KEY=noop ./node_modules/.bin/pi \
  --model llamacpp/qwen3.6-35b-a3b \
  -p "read README.md and tell me what this repo does"
```

`LLAMACPP_BASE_URL` / `OLLAMA_BASE_URL` override the default ports (`http://127.0.0.1:8888/v1` and `http://127.0.0.1:11434/v1`).

### 4. Run a benchmark

```bash
# Quick smoke
python3 benchmarks/smoke.py "What is 2+2?"

# Single Polyglot exercise
python3 benchmarks/aider_polyglot.py --exercise affine-cipher --language python --verbose

# Full Polyglot language (Python, 34 exercises)
python3 benchmarks/aider_polyglot.py --language python --resume
```

---

## Architecture (v0.1.0)

```
little-coder/
├── .pi/
│   ├── settings.json               # per-model profiles + benchmark_overrides.terminal_bench + .gaia
│   └── extensions/                 # 15 TypeScript extensions, auto-discovered by pi
│       ├── llama-cpp-provider/     # registers llamacpp/* and ollama/* as OpenAI-compat providers
│       ├── write-guard/            # Write refuses on existing files — the whitepaper invariant
│       ├── extra-tools/            # glob, webfetch, websearch (pi ships grep/find)
│       ├── skill-inject/           # per-turn tool-skill selection (error > recency > intent)
│       ├── knowledge-inject/       # algorithm cheat-sheet scoring (word=1.0, bigram=2.0, threshold=2.0)
│       ├── output-parser/          # repair malformed ```tool, <tool_call>, and bare JSON output
│       ├── quality-monitor/        # empty / hallucinated / loop detection + correction follow-up
│       ├── thinking-budget/        # cap thinking tokens per turn, retry with thinking off
│       ├── permission-gate/        # bash whitelist (ls, cat, git log/status/diff, etc.)
│       ├── checkpoint/             # snapshot files before Write/Edit
│       ├── tool-gating/            # enforces _allowed_tools on tool_call + skill filter
│       ├── turn-cap/               # max_turns abort (Polyglot unbounded, TB=25, GAIA=30)
│       ├── benchmark-profiles/     # reads settings.json → systemPromptOptions + sets temperature
│       ├── shell-session/          # ShellSession[Cwd|Reset] — tmux-proxy + subprocess backends
│       ├── browser/                # Playwright-powered Browser[Navigate|Click|Type|Scroll|Extract|Back|History]
│       ├── evidence/               # EvidenceAdd/Get/List — per-session store, 1 KB snippet cap
│       └── evidence-compact/       # preserves evidence across pi's auto-compaction
├── skills/
│   ├── tools/*.md                  # 14 tool-usage guidance files
│   ├── knowledge/*.md              # 13 algorithm cheat sheets
│   └── protocols/*.md              # 3 research/cite/decomposition workflows
├── benchmarks/
│   ├── rpc_client.py               # PiRpc — spawns `pi --mode rpc`, demuxes events/responses/UI requests
│   ├── aider_polyglot.py           # Polyglot driver, per-language transforms preserved
│   ├── tb_adapter/
│   │   └── little_coder_agent.py   # Terminal-Bench BaseAgent subclass, tmux-proxy sidecar
│   ├── gaia_scorer.py              # unchanged Python scorer
│   ├── smoke.py                    # single-prompt quick tester
│   └── test_rpc_client.py          # pytest for the RPC client
├── AGENTS.md                       # project system prompt (replaces Python context.py)
├── models.json                     # documentation-only copy of the provider registration
├── package.json, tsconfig.json, vitest.config.ts
└── docs/
    ├── whitepaper.md               # the paper — canonical version on Substack (see README top)
    ├── architecture.md             # v0.0.5-era Python architecture (preserved)
    ├── benchmark-qwen3.6-35b-a3b.md# v0.0.5 78.67% narrative
    ├── benchmark-reproduction.md   # v0.0.2 two-run reproduction
    └── benchmark-baseline-aider.md # vanilla-Aider scaffold ablation
```

**Key invariant:** pi is a minimal base (4 core tools, ~1000-token system prompt, no sub-agents / MCP / permission popups by design). Every little-coder mechanism ships as a pi extension that hooks pi's lifecycle events (`before_agent_start`, `context`, `before_provider_request`, `tool_call`, `tool_result`, `turn_end`, `session_compact`). The extensions are independent and can be enabled/disabled per deployment via `.pi/settings.json`.

---

## Reproducing the paper (v0.0.2)

```bash
git clone https://github.com/itayinbarr/little-coder.git
cd little-coder
git checkout v0.0.2
# Follow that version's README for its Python setup — pip install -e .
```

The paper ran `ollama/qwen3.5` through the Python little-coder at commit **`1d62bde`** (tag [`v0.0.2`](https://github.com/itayinbarr/little-coder/releases/tag/v0.0.2)). The 45.56 % mean figure is the average of two full 225-exercise runs on that exact codebase.

For the **78.67 % headline**, check out tag [`v0.0.5`](https://github.com/itayinbarr/little-coder/releases/tag/v0.0.5) and follow that version's llama.cpp instructions with Qwen3.6-35B-A3B.

---

## Citation

If you reference little-coder or its Aider Polyglot result in academic work, please cite the white paper:

```bibtex
@misc{inbar2026littlecoder,
  title        = {little-coder: A Coding Agent Optimized for Small Local Language Models},
  subtitle     = {Architectural Adaptation Lets a 9.7B Model Outperform Frontier Models on Aider Polyglot},
  author       = {Inbar, Itay},
  year         = {2026},
  month        = apr,
  howpublished = {\url{https://open.substack.com/pub/itayinbarr/p/honey-i-shrunk-the-coding-agent}},
  note         = {White paper}
}
```

---

## Attribution

little-coder v0.0.x was a derivative work of [CheetahClaws / ClawSpring](https://github.com/SafeRL-Lab/clawspring) by SafeRL-Lab, licensed under Apache 2.0. The upstream project provided the Python agent substrate, tool system, multi-provider support, and REPL.

little-coder v0.1.0 replaces that substrate with **pi** ([`@mariozechner/pi-coding-agent`](https://github.com/badlogic/pi-mono)) by Mario Zechner, also licensed under Apache 2.0 / MIT. The pi-mono runtime provides the agent loop, provider abstraction, TUI, and extension model; little-coder rebuilds its small-model adaptations on top of it.

All little-coder-specific mechanisms — Write-vs-Edit invariant, skill / knowledge injection, thinking-budget cap, output-parser, quality-monitor, per-model profiles, per-benchmark overrides, ShellSession/Browser/Evidence tool families, evidence-aware compaction — are preserved across versions.

---

## License

Apache 2.0 — see [LICENSE](LICENSE) for details. NOTICE tracks upstream attribution.
