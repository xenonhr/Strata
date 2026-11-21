# Arc Pro B390 runs for integrated-GPU support in the SYCL engine

The runs behind [Niko1221/Strata#6448](https://github.com/Niko1221/Strata/issues/6448) and its draft PRs: [#6453](https://github.com/Niko1221/Strata/pull/6453) (one pool), [#6455](https://github.com/Niko1221/Strata/pull/6455) (Xe3), [#6454](https://github.com/Niko1221/Strata/pull/6454) (Monitor and free memory), [#6451](https://github.com/Niko1221/Strata/pull/6451) (Swift's `--ple-gguf`) and [#6452](https://github.com/Niko1221/Strata/pull/6452) (CPU picture reader). Every file is as the scripts wrote it, except that local paths
are replaced by placeholders: `<models>` (the model's folder), `<pack>`, `<mtp>`, `<engine:NAME>` (an engine
package from the table below), `<checkout:NAME>` (a Strata checkout), `<parity-build>`, `<work>`, `<evidence>`, `~`.
`*.bin` files are float32 logit dumps, copied unchanged.

## Machine

Core Ultra X9 388H with an Arc Pro B390 (Panther Lake, `8086:b086`, `xe`), 62 GiB LPDDR5X shared with the GPU,
Fedora 44, kernel 7.2.11, compute runtime 26.35.39758. Engines built with `tools/packaging/build_sycl_linux.sh`
(engine 0.1.63) in a container with oneAPI DPC++ 2026.1.1 and oneMKL 2026.1.0.

## Model

- [`huihui-ai/Huihui-Qwen3.8-Flash-Next-abliterated-v2-GGUF`](https://huggingface.co/huihui-ai/Huihui-Qwen3.8-Flash-Next-abliterated-v2-GGUF)
  at revision `cbc7fba13f1939ac3fa42b18d77ccf43847d74ef`, folder `swift-IQ3_XXS`: a re-quant with Swift 1.5
  IQ3_XXS's two-file layout and quant types (24,576 experts, 40 GiB). SHA-256:
  - `Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf`: `e66b543e5b89f039c81e5d74c31d2410de60e3cb84a7b89aec1f2ce11fdd9830`
  - `Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf`: `d219681902f80f677b200d6bbb30387010de6cea51f6cc14ef90d873a5357a85`
- `<pack>`: `setup.py --prepare-pack <pack> --gguf-dir <models> --family swift --model IQ3_XXS`.
- `<mtp>`: the draft layer, built with `tools/mtp_fetch.py`, `tools/mtp_pack.py --experts q2_0` and `tools/mtp_rt.py`
  from `Qwen/Qwen3.8-Flash-Next`.

## Engine packages

| Name in the runs | Source |
|---|---|
| `main` | upstream main 3142b66b9 |
| `pr3729` | main + [#3729](https://github.com/Niko1221/Strata/pull/3729)'s commit (applied with `git am -3 --keep-non-patch --directory=engine`) |
| `onepool` | `pr3729` + the one-pool PR as first measured; later only comments and docs changed |
| `onepool-final` | `pr3729` + the one-pool PR's code as opened |
| `xe3onepool` | `onepool` + the Xe3 PR |
| `filldrop` | `onepool` + an experiment that drops the fill's reads from the file cache (not proposed) |

## A/B runs

`b390.json` is the run config every arm started from (no `STRATA_*` variables). Each block ran, from `terminal/`
at main 3142b66b9, with nothing else using the GPU:

    uv run --python 3.12 --with-requirements requirements.txt python tools/ab_engine.py b390.json ARM... --rounds N --log BLOCK.jsonl

An arm is `name:exe=<engine:NAME>/strata[,VAR=value...][,xargs=...]`. Per block: `BLOCK.txt` is ab_engine's report
(each round, then the medians), `BLOCK.jsonl` one line per request, `BLOCK/` the engine logs per arm and round,
`meminfo-BLOCK.log` `/proc/meminfo` once a second (MemAvailable, Cached, GPUReclaim, GPUActive, SwapFree, in kB) and
`BLOCK-oom.txt` the kernel's "Killed process" lines during the block (all empty).

| Block | Arms | Shows |
|---|---|---|
| `memory` | `main`, `pr3729`, `onepool`; 3 rounds | cache size, mirror and speed of the three builds |
| `partial-memory-1`, `partial-memory-2` | the same, stopped early by a memory watchdog on the host | earlier starts of the same builds |
| `uma0` | `main`; `onepool` with `STRATA_UMA=0`; 3 rounds | the override gives main's planning back |
| `sysman` | `pr3729` with and without `ZES_ENABLE_SYSMAN=1`; 1 round | not cited: it does not isolate Level Zero's free figure |
| `lend2` | `onepool`; `onepool` with `STRATA_LEND_MIRROR=1200,STRATA_MIRROR_MIB=2200`; 3 rounds | the lend mirror on an integrated GPU |
| `nohost` | `onepool` with `--expert-cache 16000`: `STRATA_MIRROR_MIB=0` (read from the file), a mirror, a mirror with `STRATA_VERIFY_NO_HOST=1`; 1 round | answers with experts outside the cache |
| `dropcache` | `filldrop`; `filldrop` with `STRATA_STREAM_DROP_CACHE=0`; 3 rounds | dropping the fill's reads |
| `stage` | `onepool`; `onepool` with `STRATA_PREFILL_STREAM_ALL=0`; 3 rounds | prompt staging, generic kernels |
| `stage-xe3` | the same on `xe3onepool`; 3 rounds | prompt staging, Battlemage kernels |
| `xe3-4k`, `xe3-16k` | `xe3onepool` with `STRATA_INTEL_GEN=unknown`; `xe3onepool`; 3 rounds, `--long-words 3000` / `12000` | Xe3 on the Battlemage kernels |
| `final` | `onepool-final`; 1 round | the one-pool PR's code as opened |

## Server-level runs

Served with `terminal/serve/server.py`. `hw2.txt` summarizes the runs on the `onepool` engine, `hw2-main.txt` those on
main's engine.

| Files | Run |
|---|---|
| `cfg-tight.json`, `server-tight.log`, `engine-tight.log`, `tight-answer.txt` | `onepool` with `STRATA_UMA_HEADROOM_GIB=30` and `STRATA_VERIFY_NO_HOST=1`: refused right after sizing |
| `cfg-ple*.json`, `server-ple*.log`, `engine-ple*.log`, `ple1*-answer.txt` | `--ple-gguf` naming shard 2 (`ple2`) or shard 1 (`ple1`), on `onepool`; `-main`: on main's engine |
| `cfg-reader.json`, `server-reader-*.log`, `engine-reader-*.log`, `reader-*-answer.txt` | an on-demand CPU picture reader, served from main's `terminal/` (`reader-main*`) and the reader fix's (`reader-serve-cpu-reader*`), on `onepool`; `-main`: on main's engine |
| `cfg-monitor.json`, `monitor-idle.txt`, `monitor-loaded.txt`, `server-monitor.log`, `engine-monitor.log` | the Monitor's readings and `free_vram_mib`, main against the Monitor PR, with no engine and with the model loaded on `onepool`, next to `/proc/meminfo` |

## First-token logits

`kl/`: the same token ids (`*.ids`, from the prompts in `*.json`) through the engine's one-shot mode
(`--tokens-file`, `--max-new`) with `STRATA_DUMP_FIRST_LOGITS`, on `xe3onepool`: `generic1` and `generic2` with
`STRATA_INTEL_GEN=unknown`, `bmg` with the default class. `kl2.txt` compares each dump with `generic1` using
`tools/research/tools/ab-harnesses-2026-10/arc/kl.py`.

## Builds and checks

- `parity-build.txt`, `ctest-default.txt`, `ctest-unknown.txt`: a parity build (`STRATA_SYCL_PARITY=ON`,
  `-DSTRATA_SYCL_IQ_DECODE_FAST=ON`) of the one-pool and Xe3 PRs together, and `ctest` on the card with the default
  class and with `STRATA_INTEL_GEN=unknown`.
- `parity-build-main.txt`, `ctest-main.txt`: the same for main 3142b66b9.
- `ci-xe3.txt`: CI's compile of the Xe3 PR alone (`-DSTRATA_SYCL_PARITY=OFF -DSTRATA_SYCL_IQ_DECODE_FAST=ON`).
- `check-*.txt`: `tools/check --strict` at each PR's head; `smoke-serve-*.json`: `terminal/tools/minimal_server_smoke.py --report`
  with only `terminal/requirements.txt` installed.
