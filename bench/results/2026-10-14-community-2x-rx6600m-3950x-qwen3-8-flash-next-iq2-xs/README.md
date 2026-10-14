# Community benchmark on AMD Radeon + AMD Radeon

Measured on 2026-10-14 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: AMD Radeon + AMD Radeon, 16 GiB VRAM
- CPU: AMD Ryzen 9 3950X 16-Core Processor (16 cores, 32 threads); installed RAM: 62.7 GiB; storage: not measured
- OS: Linux 7.2.10-200.fc44.x86_64; GPU driver: 7.2.10-200.fc44.x86_64; CUDA: not read
- Strata: project 0.1.44.1, commit 296b59d2; engine 0.1.44.1
- Background workloads and power limits: not recorded

## Model and configuration

- Model and quantization: qwen3.8-flash-next-iq2_xs
- Context: 32768 tokens; parallel slots: 1
- Settings: the install's own configuration, unchanged (hip backend); thinking off for the suite; temperature 0

```text
strata --pack iq2_xs --native Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf --ple-gguf Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf --expert-profile expert-profile.bin --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5 --mtp rt --max-context 32768 --kv q4_0
```

## Method

Run by `python tools/strata_bench.py` against the HTTP API of a Strata server (started by the benchmark from the install configuration); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 3 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 3 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 3 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 27.8 (23.1-30.1) | 1.12 (1.01-1.27) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 229.1 (228.0-230.3) | not measured | 17.59 (17.49-17.67) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 347.2 (326.3-349.1) | not measured | 46.31 (46.06-49.26) |
| agent session, follow-up turns | 14561 (14547-14576) at the last turn | see below | up to 64 | 3 | see below | 26.3 (22.0-31.6) | 14.64 (8.29-18.69) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 68.6 % (68.6-68.8); earlier-turn tokens read again per session: 72 (3-72) (earlier turns that the cache did not hold; the 13728 (13727-13728) new tokens of the files are read once); first turn (nothing cached) first token: 5.16 s (5.12-5.33).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 120.1 | 5.119 | 25.5 |
| 1 | 3761 | 641 | 3120 | 201.0 | 15.571 | 25.5 |
| 2 | 5294 | 3825 | 1469 | 165.1 | 8.945 | 26.7 |
| 3 | 8738 | 5357 | 3381 | 181.3 | 18.694 | 24.8 |
| 4 | 11624 | 8762 | 2862 | 199.1 | 14.432 | 27.1 |
| 5 | 14547 | 11648 | 2899 | 198.8 | 14.64 | 22.0 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load and power limits are not recorded. Total time of the suite: 518 s.
