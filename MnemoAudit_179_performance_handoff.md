# MnemoAudit performance handoff: path to `_179` and next steps

## Scope and current status

MnemoAudit is a sandboxed audit/research tool for testing deterministic weak-brainwallet derivation hypotheses against user-managed Bitcoin mainnet address datasets. The current performance work has preserved the full `fast-text-core` workload:

```text
sha256_encode, base64_encode, hex_encode, reverse, uppercase, lowercase,
rot13, atbash_cipher, leet_speak, rot47
```

The active fast route remains the current P2 scope:

```text
p2pkh_legacy, p2sh, p2wpkh
```

Taproot / `witness_v1_32` remains important for future audit coverage, but it is intentionally kept outside the current speed benchmark path until it can be added as an explicit optimized route.

The promoted release is now:

```text
Release:  _179
Build ID: mnemoaudit_v26_cuda_179_win11_batch500k_lookup_p2sh_uncomp_fastprobe_rc_v1
Stable setting: 500k source seeds per CUDA-safe batch, win11, TPB128
Validated run: 10 chunks, 100 batches, 0 OOM, 0 downshift, memory OK
Validated throughput: ~37.42–37.46M base candidates/min
```

The main reference baseline remains `_149`, at approximately **23.78M base candidates/min**. The practical target is about **47.5M/min**, equivalent to +100% over `_149`. `_179` reaches approximately **+57.5% over `_149`**, leaving roughly **+26–27% additional throughput** to reach the target.

## How we got from `_149` to `_179`

### 1. `_149`: major host-build breakthrough

Before `_149`, host-side conversion/build overhead was a dominant bottleneck. `_149` introduced the mixed native splitter:

```text
ASCII / native-eligible seeds -> native C/OpenMP builder
non-ASCII / oversized seeds  -> exact Python fallback
```

This avoided letting a small minority of non-native-compatible seeds force the whole batch through slow Python fallback. That moved the line from roughly `_146` ~12.4M/min to `_149` ~23.78M/min. This remains the key baseline because it was the first large architectural win.

### 2. `_159`–`_165`: CUDA fixed-window tuning

After `_149`, the bottleneck moved toward CUDA scalar/pubkey generation and typed HASH160 lookup. Releases `_159` through `_165` tuned the fixed-window scalar path. The best stable setting became:

```bash
--v158-cuda-scalar-threads-per-block 128
--v159-fixed-window-bits 11
--v159-fixed-window-label win11
```

This raised short-run throughput to roughly **30M/min**, but gains were diminishing by win11. Further blind window expansion was stopped.

### 3. `_168`–`_173`: stability gate and batch cleanup

Attempts to push batch size above 500k showed that 625k/700k were not stable on the RTX 3050 6 GB setup. The key discovery was that long-lived in-process CUDA runs accumulated or fragmented GPU memory across batches/chunks.

Important outcomes:

```text
625k: failed longer validation
500k without batch cleanup: eventually failed
process-isolated chunks: stable but too slow
batch-level CUDA cleanup: stable and fast enough
```

Release `_173` validated strict 500k with batch-level cleanup over 10 chunks:

```text
10/10 chunks complete
100/100 batches
0 OOM
0 downshift
~29.9M/min stable throughput
```

This created a reliable baseline for real optimization work.

### 4. `_178_2`: compressed direct-probe lookup

Profiling showed that the old compressed lookup path spent significant time preparing and sorting a shared compressed candidate array. `_178_2` added exact direct-probe lookup for compressed P2PKH and P2WPKH:

```bash
--v1782-lookup-direct-probe-compressed20
```

This removed the shared compressed preparation stage while preserving full 20-byte HASH160 exactness. The 10-chunk validation showed:

```text
~33.19M/min stable throughput
0 OOM
0 downshift
+39.6% over _149
```

This was the first major post-stability speed gain.

### 5. `_179`: fast hash/probe for P2SH and uncompressed P2PKH

After `_178_2`, the remaining lookup hot spots were:

```text
P2SH direct probe:              ~1.20s/batch
uncompressed CUDA P2PKH lookup: ~1.48s/batch
```

`_179` added:

```bash
--v179-lookup-fast-hash-probe
```

The optimized path uses C-side fast hash/probe loops:

```text
P2SH-P2WPKH:
- reuse preinitialized SHA256 state for the fixed 0x0014 redeem-script prefix
- hash candidate-specific 20-byte compressed HASH160
- RIPEMD160
- exact prefix-bounded full-20-byte lookup

Uncompressed P2PKH:
- reuse preinitialized SHA256 state for fixed 0x04 pubkey prefix
- faster coordinate conversion
- RIPEMD160
- exact prefix-bounded full-20-byte lookup
```

Validated `_179` result:

```text
10/10 chunks complete
100/100 batches
0 CUDA OOM
0 downshift
500k final batch size
memory OK on all chunks
~37.42–37.46M/min stable throughput
```

This promoted `_179` as the current best stable release.

## Rejected or deprioritized paths

The following were tested or reasoned out and should not be repeated without a clearly new hypothesis:

```text
Reducing transform count:
  Invalid as a full-workload speed fix.

RAM-prefix Python lookup:
  Catastrophic memory/runtime behavior.

GPU prefix-3 filter:
  Too saturated; only small skip rate.

GPU prefix-4 bitset:
  Too much VRAM pressure on RTX 3050 6 GB.

625k/700k batch ceilings:
  Not production-stable in this environment.

Route-family streaming from _178_1:
  Slowed throughput materially; rejected as tested.

OpenMP thread-policy A/B:
  Noise-level effect; not a meaningful lookup fix.
```

Persistent GPU workspace remains potentially useful for allocator hygiene, but the tested `_178_1` combined workspace + route-streaming path was rejected because route streaming dominated and slowed the run.

## Current bottleneck picture after `_179`

Mean per 500k-seed batch is now roughly:

```text
Total batch elapsed:          ~8.02s
Typed route lookup total:     ~2.39s
CUDA scalar GPU kernel:       ~2.26s
Host recursive build:         ~2.15s
GPU HASH160:                  ~0.047s
P2SH probe:                   ~0.677s
Uncompressed P2PKH lookup:    ~0.985s
P2PKH direct probe:           ~0.364s
P2WPKH direct probe:          ~0.359s
```

The workload is now balanced across lookup, CUDA scalar/pubkey, and host build. This is good, but it means there is no longer one obvious single bottleneck. The remaining path to the +100% target likely requires several smaller architectural wins.

## Recommended next steps

The immediate goal should be to move from **~37.5M/min** to **~40–41M/min**, then continue toward **~47.5M/min**.

Recommended next workstream order:

1. **Host-builder final-layout emission**
   - Target: reduce ~2.15s/batch host recursive build cost.
   - Idea: emit native rows directly in final seed-major/op-major layout, reducing Python merge/reconstruction overhead.
   - Expected gain: +3% to +8% if successful.

2. **CUDA scalar/output minimization**
   - Target: reduce ~2.26s/batch scalar/pubkey stage and related global-memory pressure.
   - Ideas:
     - avoid unnecessary full x/y/pubkey materialization where possible;
     - direct payload-only paths for compressed routes;
     - eventually y-parity-only compressed output.
   - Expected gain: +5% to +12%, but higher implementation risk.

3. **Persistent workspace only, without route streaming**
   - Reuse GPU buffers for `d_scalars`, `d_x`, `d_y`, `d_status`, and payload buffers.
   - This is mainly a VRAM/allocator hygiene improvement, not expected to be a large speed win alone.
   - Expected gain: +1% to +4%, plus better stability margin.

4. **Further lookup micro-optimization**
   - `_179` already reduced the biggest lookup problems, but P2SH and uncompressed lookup still cost meaningful time.
   - Only continue here after profiling `_179` carefully; avoid speculative rewrites.

A realistic remaining path is cumulative:

```text
v179 stable:                  ~37.5M/min
host final-layout improvement: +4–8%
scalar/output minimization:    +5–10%
workspace/allocator hygiene:   +1–4%
extra lookup cleanup:          +3–6%

Potential range: ~43–48M/min
```

That range overlaps the original target, but only if multiple independent optimizations land cleanly and preserve the full audit scope.
