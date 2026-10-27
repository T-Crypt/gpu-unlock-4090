# Second RTX 4090: the cleaned module on hardware, with application workloads

This is an independent repeat on a different RTX 4090, run on 2026-10-26 with the
published module at commit `4ec490d`. It answers two open items from the
[first report](../../../rtx4090-register-report.md): a hardware run of the cleaned
module, and numbers from real workloads instead of a synthetic kernel alone.

## Setup

| Item | Value |
|---|---|
| Card | GeForce RTX 4090 `10de:2684`, VBIOS `95.02.3C.40.B9`, 450 W limit |
| Host | Intel Core i9-14900K, Arch Linux, kernel `7.2.11-arch1-1` |
| Driver | unmodified NVIDIA `615.78.08` |
| Display | Intel iGPU; the 4090 drives no screen |
| Module | `gpu-unlock-dkms` and `gpu-unlock-firmware` 1.0.0-1 from `packaging/arch`, DKMS, mkinitcpio UKI |
| Command line | `gpu_unlock.targets=0000:01:00.0` |

We took every "before" number on the same boot setup with the package not yet
installed, then powered the machine off, powered it on, and repeated each run.

## Result

The cold boot logged `pci 0000:01:00.0: gpu_unlock: unlocked` and the stock driver
brought the card up as usual ([kernel-log.txt](kernel-log.txt)).

### Tensor microbenchmark

`diagnostics/bench/mma_bench.cu`, `nvcc -O3 -arch=sm_89`, 20,000 iterations, best of three.

| Mode | Before | After |
|---|---:|---:|
| FP16 x FP16, FP32 accumulate | 165.3 | **330.5** |
| BF16 x BF16, FP32 accumulate | 165.3 | **329.7** |
| FP16 x FP16, FP16 accumulate | 330.5 | 315.3 |
| INT8, INT32 accumulate | 660.7 | 659.2 |
| `dp4a` | 163.9 | 163.9 |
| scalar `ffma` | 78.8 | 79.0 |

FP8 with FP32 accumulate, measured with the FP8 mode from
[JustAResearcher/gpu-unlock-ada](https://github.com/JustAResearcher/gpu-unlock-ada)'s
`mma_bench.cu` (main `d28c485`) on the unlocked card, read **655.0 TF/s** in all three
runs ([mma-unlocked-fp8.txt](mma-unlocked-fp8.txt)). We did not run FP8 before the
unlock. NVIDIA rates a stock RTX 4090 at 330.3 TF/s for FP8 with FP32 accumulate and
660.6 TF/s with FP16 accumulate, so this card now issues FP8 at 99% of the full rate.

The FP16-accumulate row swings between 306 and 331 TF/s from run to run on both
sides, so its change is noise. The rates match the first report within 0.5%.

After the LLM and ComfyUI runs below had each loaded and unloaded their engines on
the card, a fresh run still read 327.4 / 330.0 TF/s
([mma-after-engine-swaps.txt](mma-after-engine-swaps.txt)). Runtime D3 is off on
this desktop, so the card kept power the whole time.

### LLM inference: no gain

A custom CUDA engine serving a 27B Qwen3.8 model (4/5-bit group-wise weights, INT8
activations in prefill, speculative decoding), mean of two runs per size:

| Prompt | Prefill before | Prefill after |
|---|---:|---:|
| 7.3K tokens | 3,655 t/s | 3,678 t/s |
| 60K tokens | 3,017 t/s | 3,032 t/s |
| 245K tokens | 1,654 t/s | 1,660 t/s |

Decode stayed within noise (77 to 139 t/s depending on context). A greedy 256-token
reply came out byte-identical before and after. The engine runs its prefill GEMMs as
INT8 MMA and decode is bound by memory bandwidth, so neither path issues the FP16/BF16
FP32-accumulate instructions the unlock speeds up.

llama.cpp (build b11716, `-fa on`, q8_0 KV cache) measured after the unlock only, so
the table below has no "before" yet:

| Model | 7.4K prefill | 60K prefill | 122K prefill |
|---|---:|---:|---:|
| Qwen3.6-35B-A3B UD-IQ4_XS (MoE) | 9,348 t/s | 8,020 t/s | 6,305 t/s |
| Qwen3.8-27B IQ4_XS (dense) | 3,148 t/s | 2,665 t/s | 2,161 t/s |

Raw rows: [llamacpp-unlocked.jsonl](llamacpp-unlocked.jsonl). Quantized llama.cpp
matmuls run through INT8 MMA (MMQ), so we expect little change there too. A boot
without the unlock will fill in the missing column.

### Diffusion: 1.20x

ComfyUI, Flux.1 Krea dev (fp8 e4m3 weights), 1024x1024, 30 steps plus a refine
pass at 0.35 denoise, three warm runs with different seeds:

| | Before | After |
|---|---:|---:|
| Warm generation | 21.9 / 21.9 / 21.9 s | 18.3 / 18.2 / 18.3 s |
| First run with model load | 61.8 s | 54.1 s |

Same-seed images match pixel for pixel before and after, outside the film-grain band
described in the notes below; that band changes from run to run on its own
([comfyui-flux-timings.txt](comfyui-flux-timings.txt)).

## Notes for anyone repeating this

- If your ComfyUI output shows noise or a text-like band after the unlock, check a
  baseline image first. The ComfyUI Film Grain GLSL blueprint declares only
  `precision highp float;`, so ANGLE on Vulkan computes its `uint` hash at mediump
  precision and corrupts pixels on this card with or without the unlock. Adding
  `precision highp int;` fixes it.
- This fork has no suspend notifier. After suspend to RAM the card runs at stock
  rates until the next cold boot.

## Files

| File | Contents |
|---|---|
| [kernel-log.txt](kernel-log.txt) | `gpu_unlock` lines from the unlocked boot (root device redacted) |
| [mma-baseline.txt](mma-baseline.txt), [mma-unlocked.txt](mma-unlocked.txt), [mma-after-engine-swaps.txt](mma-after-engine-swaps.txt) | `mma_bench` stdout |
| [mma-unlocked-fp8.txt](mma-unlocked-fp8.txt) | gpu-unlock-ada `mma_bench` with the FP8 mode, unlocked only |
| [ninfer-baseline.jsonl](ninfer-baseline.jsonl), [ninfer-unlocked.jsonl](ninfer-unlocked.jsonl) | LLM engine timing rows and the parity reply |
| [llamacpp-unlocked.jsonl](llamacpp-unlocked.jsonl) | llama.cpp timing rows, unlocked only |
| [comfyui-flux-timings.txt](comfyui-flux-timings.txt) | Flux wall-clock timings |
