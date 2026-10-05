# Qwen3.8-Flash-Next on a single AMD Radeon AI PRO R9700 (32 GB) — Windows 11 + HIP

Real-world, long-run data for running **Qwen3.8-Flash-Next (IQ2_XS)** on **one AMD
Radeon AI PRO R9700** under **Windows 11** with the **HIP** backend, via
[Strata](https://github.com/Niko1221/Strata).

This is a data-sharing repo, not a guide: we publish what we measured so the next
person with this card does not have to rediscover it, including the dead ends.

> **Status:** working and in daily use since 2026-10-02. 3-day soak: **396/402
> requests OK (98.5%)**, MTP acceptance **1.00**.

---

## 1. The question this answers

When we first searched for "R9700 + Qwen3.8-Flash-Next", the public evidence was
thin: the Strata README's verified list stopped at the **RX 9070 XT**, and the
R9700 (Navi 48, `gfx1201`, 32 GB) is a different board with a different VRAM
budget story on Windows.

Since then several reports have appeared — see §7. **This repo exists to add a
long-duration data point**, not to claim a first.

---

## 2. Hardware / software

| | |
|---|---|
| GPU | AMD Radeon AI PRO R9700, 32 GB, `gfx1201` (RDNA 4, Navi 48) |
| OS | **Windows 11** |
| Backend | **HIP** (`backend: "hip"`, `lib_dirs: engine/rocm/bin`) |
| Engine | Strata **v0.1.34** — *behind current; see §5* |
| Quant | Qwen3.8-Flash-Next **IQ2_XS** (GSQ-RCO, 2-shard GGUF + PLE) |
| Context | 131,072 (`--max-context`) |
| KV | `--kv int8`, `--kv-resident 32768` |
| Speculation | `--spec 4`, `--spec-min-p 0.5`, MTP |
| Other | `--expert-cache auto`, `--prefill auto`, `--expert-profile data/expert-profile.bin` |
| Tuning env | `STRATA_HIPBLASLT_TUNING=tools/hip/gfx1201-hipblaslt-100500.txt` |

Exact argument list (from the live process config):

```
strata.exe --pack <packs/iq2_xs> --native <...-00001-of-00002.gguf>
  --ple-gguf <...-00002-of-00002.gguf> --expert-profile data/expert-profile.bin
  --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5 --mtp <mtp/rt>
  --max-context 131072 --kv int8 --kv-resident 32768
```

Served through `serve/server.py --engine strata --config <cfg>.json --port 8081`.

---

## 3. Measured throughput

### 3.1 Long-run soak (the headline number)

A fixed short prompt, 96 output tokens, **every 10 minutes for ~3 days**
(2026-10-02 14:15 → 2026-10-05 08:00 CST). Raw aggregate in `soak_stats.json`.

| Metric | Value |
|---|---|
| Requests | **402** (396 OK → **98.5%**) |
| Decode median | **66.4 tok/s** |
| Decode mean | 64.3 tok/s |
| Decode p10 / p90 | 34.6 / 98.7 tok/s |
| Decode min / max | 19.3 / **112.6 tok/s** |
| Prefill median | **54.9 tok/s** |
| Prefill max | 149.6 tok/s |
| **MTP acceptance** | **1.000** (median and mean) |

**Reading this honestly:** the soak ran on a **shared desktop** — it is a
*mixed-condition* number, not a quiet-machine benchmark. The recent
steady-state samples (§3.2) show **~73–92 tok/s** when the machine is otherwise
idle. So:

- **Quote 66 tok/s** as "typical on a working desktop".
- **Quote ~90 tok/s** as "idle machine, warm".
- **113 tok/s** is the observed ceiling — do not present it as typical.

Anyone quoting only the peak is cherry-picking. We publish the p10 (35) for the
same reason: it is what you get when the box is busy.

### 3.2 Sustained single-stream, warm

Recent steady-state samples (server-reported timings):

```
decode 92.0 tok/s   prefill 82.4 tok/s   mtp 1.0
decode 73.0 tok/s   prefill 76.9 tok/s   mtp 1.0
decode 92.2 tok/s   prefill 69.2 tok/s   mtp 1.0
```

### 3.3 Long-context

From our own runs and the PR #618 dataset (§7), long context degrades
gracefully rather than falling off a cliff:

| Prompt tokens | Decode tok/s |
|---|---|
| ~4 K | ~100 |
| ~33 K | ~98 |
| ~131 K | ~90 |
| ~248 K | ~88 |

### 3.4 Quality

A 51-question structured benchmark (deterministic JSON scoring):

| Category | Score |
|---|---|
| Financial reasoning | **96.8%** |
| General | 81.8% |
| **Overall** | **92.3%** |

Tool calling was the weak spot (~55.6%); general reasoning ~63.6%. Treat this
as a capable analyst for long-context financial reading, **not** as an
agentic tool-caller.

---

## 4. Issues we hit and how they were handled

### 4.1 `hipBLASLt launch failed with status 6` — persistent, non-fatal

**This is the most useful thing in this repo** because it looks alarming and
does not appear in the vendor docs.

- **4,160 occurrences** in a single ~3-day engine log (≈60% of all log lines).
- It appears from the very first minutes of a run and continues throughout —
  not a startup-only condition.
- **Despite the noise, throughput is unaffected**: the same log holds 804
  timed generations with median 83.6 tok/s and a 127.8 tok/s peak. The call
  evidently **falls back** to another GEMM path.
- We do **not** yet have a root cause. If you see it, it is probably not why
  your speed is bad — measure first.

### 4.2 MTP is worth keeping

Acceptance is **1.000** across 396 samples. If you are tuning, do not disable
MTP to "avoid speculation overhead" on this workload; there is nothing to lose.

### 4.3 Windows-specific: VRAM budget vs system RAM

A Windows-specific fix landed upstream in **v0.1.37**:

> the engine counts the desktop's VRAM (WDDM budget), so the expert cache no
> longer spills into system RAM

Reported effect on an RX 6800: **31 → 42 tok/s**. **We are still on v0.1.34 and
have not applied this** — see §5, it is our top pending item.

---

## 5. What we would do differently (honest gaps)

1. **Upgrade the engine.** We are on **0.1.34**; upstream is at **0.1.39**. We are
   missing: the **v0.1.37 Windows VRAM-accounting fix** (§4.3), **v0.1.36**
   IQ2_XS prompt-path gains (+12% at 4K, +3% at 32K), and the **v0.1.35–0.1.39**
   AMD/Windows fixes generally. Any tok/s figure here should be read as a
   **floor**, not a ceiling, for that reason.
2. **Report the distribution, not the peak.** Our early internal notes recorded
   a single 111 tok/s figure that was actually a warm prefill number; the soak
   median is 66. Beware of mixing prefill and decode in the same sentence.
3. **Do not put this node in a cron/scheduler** if the machine is also a
   desktop — it gets shut down and rebooted for other work. We explicitly keep
   it out of scheduled jobs.

---

## 6. Caveat: the Windows VRAM budget

The card is a **discrete** 32 GB board — but on Windows you do not get to treat
all 32 GB as model budget. The **WDDM budget** (what Windows will let a process
allocate, accounting for the desktop compositor and other clients) is the number
the engine has to plan around, and it is smaller than the sticker figure.

This is precisely the class of bug fixed in **v0.1.37**, whose notes explain that
the engine had been failing to count the desktop's VRAM, causing the expert cache
to spill into system RAM. If you are on an older build and your cache behaviour
looks inexplicable, this is the first thing to check.

Practical advice: budget conservatively, and prefer measuring the engine's own
reported free/total over assuming the sticker number.

---

## 7. Community reports (credit where due)

| Where | What |
|---|---|
| [Strata PR #618](https://github.com/Niko1221/Strata/pull/618) | **R9700 + Windows 11 + IQ2_XS**, v0.1.38: 99.8 tok/s @4K → 87.5 @248K, **needle recall 6/6**, no stalls. Closest match to our setup. |
| [Strata PR #178](https://github.com/Niko1221/Strata/pull/178) | R9700 on ROCm 7.14 / Ubuntu: 62.7 prefill, 44.7 decode, MTP 85.7%. Needed a small patch to accept `gfx1201` under a `gfx1100` guard. |
| [Strata PR #262](https://github.com/Niko1221/Strata/pull/262) | `gfx1201` support: Q2_0 decode **58 → 104 tok/s** — `__dp4a` had been falling back to a bytewise loop off-gfx1100. |
| [Strata #697](https://github.com/Niko1221/Strata/issues/697) | Windows+HIP stall at `GPU rang 0`, 0 layers served → add `__threadfence_system()` to `doorbell_publish_kernel`. |
| [Strata #541](https://github.com/Niko1221/Strata/issues/541) | R9700/Linux: batched prompt read stalls; `STRATA_PLE_BATCH=0` works around it. |

If you have an R9700 result, upstream's issue tracker is where it belongs —
this repo is just a mirror of our measurements.

---

## 8. Reproduce

```bash
# 1. Strata (Windows + HIP build), then:
serve/server.py --engine strata --config strata-iq2_xs.json --port 8081

# 2. probe (any OpenAI-compatible client)
curl -s http://127.0.0.1:8081/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3.8-flash-next-iq2_xs",
       "messages":[{"role":"user","content":"1,2,3,4,5,6,7,"}],
       "max_tokens":96}'
```

To reproduce the soak: same request every 600 s, record
`timings.predicted_per_second` (decode) and `prompt_per_second` (prefill) from
the **server** response, not client wall-clock — client wall time includes
queueing and will understate throughput.

---

## 9. Files

- `soak_stats.json` — aggregate statistics over the 402-request window.

---

## License / attribution

Measurements are ours, published as-is, no warranty. Strata and
Qwen3.8-Flash-Next are the work of their respective authors; this repo claims
nothing about them and is not affiliated.
