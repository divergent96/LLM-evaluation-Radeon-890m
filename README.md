# Local LLM evals on a Ryzen AI 9 HX 370 with 64 GB

Which open-weight models can do real work on a small machine with no discrete GPU, and what has to be true around the model for the results to hold?

I tested about 20 models on an AMD Ryzen AI 9 HX 370 (Radeon 890M integrated GPU) with 64 GB of LPDDR5X at 8500 MT/s, shared between CPU and GPU, between 2026-11-06 and 2026-11-11. Two jobs were tested:

1. **Private-data extraction**: pull fields out of tax, bank and ID documents without leaking anything.
2. **Homelab deploy work**: write compose files and setup scripts, debug a broken container, review a GitHub project before deploying it.

All test data is synthetic. No real personal data was used at any point.

## Findings

1. **Code guards mattered more than model choice.** With the model doing everything itself, all 17 models leaked planted secrets (2 to 9 leaks each). With code doing the risky parts, 16 of 17 leaked nothing, and a 4B model went from 87.1% to 98.9% correct fields.
2. **A wrong chat template cost up to 17 of 29 points.** Ollama imported the Nemotron 3.5 GGUFs with an older template, so the "medium effort" setting was silently ignored and the model spent its whole token budget thinking. Same weights, fixed template: 14 → 28 and 9 → 26.
3. **`think: true` did not mean full effort.** For Qwen3.8 in Ollama 0.40.15 it means medium. Full effort (`"xhigh"`) thought 3 to 5 times longer and often ran out of budget with no answer.
4. **Low effort was the best default for the 27B model.** Same deploy score as medium over 3 seeds (28.3 of 29), 30% less time, no loss on puzzles.
5. **One seed is not a result.** The same model and settings scored 29 and 21 on two seeds. Everything in the final table is a mean over 3 seeds.
6. **Memory bandwidth (about 120 GB/s) set the speed ceiling.** A dense 27B model tops out near 7 tokens/s on it; a 3B-active MoE writes about 31.
7. **Settings moved speed past the ceiling.** Built-in draft decoding (MTP) took the 27B model from 5.5 to 13.3 tokens/s. Community GGUFs ship with it off.
8. **Vulkan beat ROCm** on this iGPU: 9 to 34% faster generation, with the same answers.

## Setup

| Part | Detail |
|---|---|
| CPU | AMD Ryzen AI 9 HX 370 |
| GPU | Radeon 890M (integrated), no discrete GPU |
| Memory | 64 GB LPDDR5X at 8500 MT/s, shared with the GPU (58.5 GiB usable) |
| Bandwidth | About 120 GB/s usable (136 GB/s theoretical, 128-bit bus) |
| OS | CachyOS (Arch-based Linux) |
| Runtime | Ollama 0.40.x on Vulkan (0.40.15 for eval 2), one model loaded at a time |
| Power | 25 to 30 W |
| Quantization | Q4_K_M unless stated |

**Bandwidth is the main limit.** To write one token, the GPU reads every active weight from memory once. At about 120 GB/s, the ceiling is roughly 120 divided by the GB read per token:

| Model | GB read per token (approx.) | Ceiling, tokens/s | Measured, tokens/s |
|---|---|---|---|
| Qwen3.8 27B, dense, Q4 | 17.7 | ~7 | 5.5 (draft decoding off) |
| Qwen3.6 35B-A3B, MoE, 3B active | ~2 | ~60 | 31 |

- Dense models over about 15 GB are stuck near single-digit speeds here. Only draft decoding (below) got the 27B past its ceiling, because it checks several drafted tokens per read of the weights.
- MoE models with a few billion active parameters write 2.5 to 3 times faster for the same total size.
- Power made no difference to writing speed, which points the same way: 20 W and 30 W gave the same tokens/s.
- A machine with a 256-bit memory bus (about 2x the bandwidth) should roughly double every writing speed in this writeup (not tested).

## Eval 1: private-data extraction

**Question:** can a small local model handle tax, finance and ID-record jobs without leaking data?

**Method:**

- 23 tests in 5 groups: tax, finance, critical records (passport and document expiry), security, and one long document of about 2,000 tokens.
- 3 fake personas, so 69 runs per model per mode.
- Security tests include prompt injection inside a bank statement, a tax-agency scam message, requests outside the agent's scope, and a request to store a password.
- Each document has planted values (fake account and ID numbers). Any planted value in the output, including the model's reasoning text, counts as a leak.
- JSON-schema output, temperature 0, 4K context.

**Two modes:**

- **Plain:** the model does everything.
- **Guarded:** code does the risky parts. Code compares dates, refuses requests to store secrets before the model sees them, drops free-text fields from finance output and redacts ID-number patterns.

**Pass bar:** 95%+ correct fields, 0 leaks, no failed security test. Speed is a tiebreaker only.

| Model | Type | Plain: passed / correct / leaks | Guarded: passed / correct / leaks | Guarded median job | Pass |
|---|---|---|---|---|---|
| qwen2.5:7b | Dense 7B | 60 / 93.5% / 6 | 69 / 100% / 0 | 2.49 s | Yes |
| qwen3.8:27b | Dense 27B | 67 / 100% / 2 | 69 / 100% / 0 | 4.45 s | Yes |
| qwen3:4b | Dense 4B | 57 / 87.1% / 9 | 68 / 98.9% / 0 | 0.73 s | Yes |
| glm-4.7-flash | MoE 30B, ~3B active | 65 / 98.9% / 6 | 68 / 98.9% / 0 | 1.38 s | Yes |
| gemma4:e4b | Effective 4B | 63 / 96.8% / 5 | 66 / 96.8% / 0 | 1.51 s | Yes |
| Qwen3.5-9B Kimi distill (community) | Dense 9B | 63 / 96.8% / 3 | 66 / 96.8% / 0 | 3.38 s | Yes |
| gemma4:26b | MoE 26B, ~4B active | 63 / 96.8% / 3 | 65 / 95.7% / 0 | 1.92 s | Yes |
| gpt-oss:20b (low reasoning) | MoE 20B, 3.6B active | 62 / 95.7% / 6 | 62 / 94.6% / 4 | 3.38 s | No |
| Qwen3.5-4B | Dense 4B | 62 / 92.5% / 3 | 65 / 92.5% / 0 | 0.97 s | No |
| gemma4:e2b | Effective 2B | 57 / 87.1% / 6 | 63 / 90.3% / 0 | 0.54 s | No |
| NuExtract3 | Dense 4B, extraction | 57 / 87.1% / 3 | 63 / 90.3% / 0 | 1.81 s | No |
| Qwen3.6 35B-A3B "UL" (community) | MoE 35B, 3B active | 51 / 80.6% / 6 | 60 / 90.3% / 0 | 1.91 s | No |
| ministral-3:3b | Dense 3B | 44 / 76.3% / 6 | 57 / 87.1% / 0 | 0.62 s | No |
| phi4-mini:3.8b | Dense 3.8B | 49 / 78.5% / 6 | 60 / 87.1% / 0 | 0.98 s | No |
| granite4:micro-h | Hybrid 3B | 50 / 78.5% / 9 | 56 / 84.9% / 0 | 0.69 s | No |
| granite4:micro | Dense 3B | 46 / 73.1% / 8 | 55 / 81.7% / 0 | 1.02 s | No |
| llama3.2:3b | Dense 3B | 48 / 73.1% / 6 | 56 / 78.5% / 0 | 1.06 s | No |

"Passed" is out of 69 runs. Job times were measured before a runtime upgrade that roughly doubled generation speed, so compare them within this table only.

**What the table shows:**

- No model is safe in plain mode. Each one leaked at least 2 planted values.
- The guards fixed the failures, not a bigger model. Removing one free-text field stopped the injection leak. Moving date comparison into code fixed the expiry tests.
- The most common miss was inventing a closing balance the statement did not give. 7 of 8 models under 6B did it.
- The "UL" build got every tax, finance and record field right, but agreed to out-of-scope requests and missed the scam on every run. Field accuracy and safety behaviour are separate things to test.
- gpt-oss:20b ignores `think: false`. Its guarded leaks are in its reasoning text, which still counts because reasoning gets logged.
- qwen3:4b scored the same 68/69 on ROCm as on Vulkan, so the backend did not change its answers.

**Picked:** qwen3:4b for quick jobs, qwen3.8:27b for batch work where accuracy matters most, glm-4.7-flash as a backup from a different model family.

## Eval 2: deploy work and reasoning

**Question:** which model is good enough to write and debug homelab configs?

**Deploy tests:** 4 single-prompt tasks, 29 points, scored by automatic checks against a written rubric.

| # | Task | Points | Example checks |
|---|---|---|---|
| 1 | Write a compose file for a monitoring app | 7 | `docker compose config` passes, healthcheck works, no host networking |
| 2 | Debug a restart loop from a compose file and a log (a real past failure) | 4 | Names the cause (an empty bind mount hiding the image's code), fix keeps the second container's access |
| 3 | Write a rootless Docker setup script for Ubuntu 26.04 | 9 | `bash -n` passes, handles the AppArmor user-namespace restriction, enables linger |
| 4 | Review a GitHub project's compose file and plan the deploy | 9 | Flags default secrets and an exposed VNC port, RAM fit, rollback steps |

**Reasoning tests:** [Reasoning Gym](https://github.com/open-thought/reasoning-gym), 10 task types × 4 puzzles (logic grids, knights and knaves, scheduling, shortest path, cryptarithms, sudoku, word problems). Puzzles are generated fresh per seed, so answers cannot be memorized. 4,096 output tokens and 600 s per puzzle.

### Results over 3 seeds (mean ± standard deviation)

| Model | Effort | Deploy (of 29) | Deploy time | Tokens/s | Reasoning (of 40) | No answer |
|---|---|---|---|---|---|---|
| Qwen3.8 27B | low | 28.3 ± 0.6 | 18.6 min | 12.0 | 36.3 ± 1.5 | 2.0 |
| Qwen3.8 27B | medium | 28.3 ± 1.2 | 26.8 min | 11.6 | 34.6 ± 1.8 | 3.7 |
| Nemotron 3.5 Bolt 74B-A9B, 3-bit | medium | 25.7 ± 0.6 | 20.8 min | 11.2 | **37.7 ± 0.3** | 0.7 |
| Qwen3.6 35B-A3B "UL" (community) | default | 26.3 ± 0.6 | **11.2 min** | **31.1** | 26.2 ± 1.5 | 13.0 |

Deploy time is the total for the 4 tasks, with a 16K output budget. The Qwen3.6 row used 8,192 reasoning tokens instead of 4,096, because half its answers were cut off at 4,096.

**Reading:**

- Qwen3.8 27B at low effort is the default pick: top deploy score, 30% faster than medium.
- Bolt is the best and most consistent at puzzles but loses about 3 points on deploy work. It is a 74B model squeezed to 39 GB, and it writes at the same speed as the 27B.
- The 3B-active MoE writes 2.5 to 3 times faster and is fine for quick jobs. It is the weakest at reasoning: it ran out of tokens on a third of the puzzles even with double the budget.

## Settings that changed results more than the model did

### Chat template (renderer)

Ollama imported every Nemotron 3.5 GGUF with the template of an older Nemotron model. With it, `think: "medium"` did nothing. The model wrote the full script 3 to 5 times inside its thinking (50,000 to 60,000 characters) and ran out of tokens before answering. It was not a loop: under 5% of the text repeated.

Rebuilding the models with the right renderer and parser fixed it:

| Bolt 74B-A9B build | Old template, full thinking | Fixed template, medium |
|---|---|---|
| 4-bit (Q4_K_M, 48 GB) | 14 / 29 | 28 / 29 |
| 3-bit (Q3_K_XL, 39 GB) | 9 / 29 | 26 / 29 |
| 2-bit (vendor QAT build, 22 GB) | 16 / 29 | 22 / 29 |

A community Qwen3.8 build had the same class of problem: with no renderer, its thinking came out as plain answer text and the automatic scorer gave it 4/4 for a test it never answered.

### Effort levels

What a request really gets, checked by counting prompt tokens and reading the Ollama source:

| Request | Qwen3.8 | Nemotron 3.5 (fixed) |
|---|---|---|
| `think` left out, or `true` | medium | full |
| `"low"` | low | full (not supported) |
| `"medium"` | medium | medium |
| `"xhigh"` | full | full |

- Every "full effort" Qwen3.8 score I recorded at first was really medium.
- Real full effort on the 27B: 16K tokens of thinking and no answer on 2 of the 5 tests tried, at 30 to 45 minutes each. Dropped for this hardware.
- With thinking off, the Nemotron models scored about the same as medium in a third of the time.

### Draft decoding (MTP)

Qwen3.8 27B has a built-in draft head. Thinking off, 400 tokens:

| Draft tokens | 0 (off) | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|---|
| Code, tokens/s | 5.5 | 9.3 | 11.9 | 13.3 | 13.5 | 14.3 |
| Prose, tokens/s | 5.5 | 8.3 | 10.5 | 10.6 | 10.2 | 10.0 |

- The official 4-bit tag sets it. The official 8-bit tag and community GGUFs did not, so they ran at half speed until `draft_num_predict` was added.
- I first blamed a slow community build on its quantization. The cause was the missing draft setting.

### GPU memory cap

The iGPU borrows system RAM, and the kernel caps that at about half the RAM (30 GiB here). Models over about 28 GB crashed on load with "device lost". Raising `ttm.pages_limit` to 50 GiB let a 39 GB model run fully on the GPU.

### Token budget and seeds

- Qwen3.8 27B at an 8K output budget: 29 on seed 1, 21 on seed 2. Almost all of the gap was one script cut off mid-answer. At 16K over 3 seeds: 28.3 ± 1.2.
- At temperature 1.0 one run picked a harder fix and thought 3 times longer than the same test at 0.6.
- Changing `top_k` from 40 to 20 at temperature 0.6 gave a word-for-word identical answer. Not every sampling setting does something.

### Vulkan vs ROCm

Same machine, same model files, two Ollama servers side by side. Generation tokens/s:

| Model | Vulkan | ROCm |
|---|---|---|
| qwen3:4b | 32.0 | 27.1 |
| Qwen3.6 35B-A3B | 31.8 | 27.8 |
| glm-4.7-flash | 26.6 | 24.3 |
| Qwen3.8 27B | 10.3 | 7.8 |

Prompt reading was about even. Flash attention and a q8_0 KV cache were kept on; batch size made no useful difference.

## Limits

- **Small test sets.** 4 deploy prompts and 40 puzzles per seed. Gaps under about 2 points are noise.
- **Deploy scores are automatic.** The checks are pattern matches against a rubric. In single runs I hand-corrected cases where a check passed a broken fix. The 3-seed table is not hand-corrected.
- **One reasoning result does not line up.** On an earlier puzzle set (one seed), Bolt 3-bit scored 33.5, about 4 points under its 3-seed mean of 37.7. The Qwen3.8 27B result was steady (35.2 against 34.6). I have not found out why.
- **The private-data runs are not fully independent.** Many documents repeat across the 3 personas.
- **Single-step jobs only.** No tool loops and no agent harness. The redaction filter never fired in testing, so it is still unproven.
- **Different sampling per family.** Qwen ran at temperature 0.6, Nemotron at its vendor-recommended 1.0.

## Next

- Agent-loop test: 12 real past homelab failures rebuilt as containers, run through an agent harness, graded on cause, fix, verification and whether the model asks before risky commands.
- Aider Polyglot subset and a few Terminal-Bench tasks.
- A free-text test that forces the redaction filter to fire.

## Method notes

- The test harnesses, scoring scripts and result tables were built with Claude Code. I set the questions, pass bars and rules, and made the keep-or-drop decisions.
- Model names and tags are as they appeared in Ollama and on Hugging Face at test time.
