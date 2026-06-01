# Prompt to start a new MnemoAudit development chat from release `_167`

Copy-paste the full prompt below into a new chat. Upload the latest MnemoAudit release files and recent result ZIPs when available.

---

I am continuing development of **MnemoAudit**. Please read this carefully and do not misunderstand the project scope.

## Project scope and safety framing

MnemoAudit is a **sandboxed audit/research tool** for testing deterministic weak-brainwallet derivation hypotheses against a database of **managed Bitcoin mainnet addresses generated specifically for this research**.

It is **not malware**, not a wallet stealer, not a recovery tool, and must never be framed as attacking Bitcoin. It is a controlled experiment over user-managed address datasets and deterministic weak-seed hypotheses.

The performance target is to make a workload that currently takes about **1 hour** finish in about **30 minutes**. In engineering terms, the target is **+100% throughput relative to release `_149`**.

Do **not** reduce the audit scope as a real speed fix. Reduced derivation suites may exist only as explicitly scoped experiments. The main full-workload target remains the full `fast-text-core` 10-transform workload:

```text
sha256_encode, base64_encode, hex_encode, reverse, uppercase, lowercase,
rot13, atbash_cipher, leet_speak, rot47
```

Never claim performance improved unless uploaded benchmark reports prove it.

## Hardware and environment

Target/actual environment:

```text
OS: Ubuntu 26.04 LTS target
CPU: Intel i7-class CPU
RAM: 16 GB currently; 32 GB expected soon
GPU: NVIDIA RTX 3050 6 GB
RustDesk: often active during tests
Project root: /run/media/dario/500/python/btc/.venv/MnemoAudit
Seed-bin root: ./mnemoaudit_wordlist_25gb_v100/seedbins
Historical scan root: ./mnemoaudit_wordlist_25gb_v100/scans_118_fast_text_core
Managed DB: ./managed_addresses.sqlite
Typed lookup DB: ./managed_addresses_typed_lookup.sqlite
Typed bin dir: ./managed_typed_bin
```

Current address scope for fast P2 route:

```text
--typed-include-address-types p2pkh_legacy,p2sh,p2wpkh
--exclude-taproot-fallback
--exclude-uncompressed-p2pkh-fallback
--test-profile p2-cuda-full
```

Taproot / `witness_v1_32` remains important for future research coverage, but must be added as an explicit route later, not by enabling a slow fallback.

## Critical scan-root policy

Use the historical scan root **only** for continuing the existing 439-chunk campaign:

```text
./mnemoaudit_wordlist_25gb_v100/scans_118_fast_text_core
```

Use **fresh isolated scan roots** for A/B tests and stability validation, so partial resume state does not skew benchmark interpretation.

Good examples:

```text
./mnemoaudit_wordlist_25gb_v100/scans_168_win11_625k_10chunk
./mnemoaudit_wordlist_25gb_v100/scans_168_lookup_ab_variant_a
./mnemoaudit_wordlist_25gb_v100/scans_168_lookup_ab_variant_b
```

Do not accidentally create new production roots when the goal is to resume the historical campaign. Do not accidentally use the historical campaign root when the goal is clean benchmarking.

## Important operational rule

Every release/test command must create **one upload-ready root-level ZIP** so I do not have to search across `scan_logs`, `scan_reports`, `chunk_*`, and prewarm folders.

Preferred pattern:

```text
./mnemoaudit_reports_temporary_<release>_<test_name>/
./mnemoaudit_reports_temporary_<release>_<test_name>.zip
```

At the end of every benchmark script, print only the ZIP(s) I must upload.

Use `py_compile` without creating local `__pycache__` clutter:

```bash
python3 - <<PY
import py_compile
py_compile.compile("$SCRIPT", cfile="/tmp/mnemoaudit_pycompile_check.pyc", doraise=True)
print("py_compile OK:", "$SCRIPT")
PY
```

Do not automatically provide these files unless I explicitly ask for them:

```text
mnemoaudit_*_cuda_build_info.txt
PROMPT_FOR_CODING_LLM_MnemoAudit_v*.md
README_MnemoAudit_v*.md
RELEASE_NOTES_MnemoAudit_v*.md
```

But every release reply should include:

```text
versioned filename
build ID
short changelog
py_compile check
--cuda-build-info check
exact diagnostic command
exact benchmark command when appropriate
expected metrics
honest statement of what was not improved
copy-paste bash command for bug check and performance benchmark
```

## Baseline and target

Important baseline:

```text
_149 mixed native splitter best 2-chunk throughput: ~23.78M base candidates/min
Double-speed target: ~47.5M base candidates/min
Current short-run best after latest work: ~31.0M/min
Current improvement over _149: roughly +30% to +31%
Remaining gap to target: large; needs architectural improvements, not small tuning only
```

## Key history and findings

### Big win already achieved

`_149` solved the major host-build fallback problem by using a mixed native splitter:

```text
ASCII / native-eligible seeds -> native C/OpenMP builder
non-ASCII / oversized seeds  -> exact Python fallback
then merge back to original seed-major/op-major order
```

This moved performance from roughly `_146` 12.4M/min to `_149` around 23.8M/min and reduced host-build dominance.

### Current stable route shape

After `_149+`, the main bottlenecks are broadly:

```text
CUDA scalar/pubkey: large but improved by fixed-window tuning
HASH160 lookup: still significant
Host recursive build: still significant but no longer dominant
GPU HASH160: negligible
```

### Failed or weak directions

Do not waste time repeating these unless there is a clearly new hypothesis:

```text
- Reducing transforms is not a valid full-workload speed fix.
- Dedup host builder did not materially improve this dataset.
- Zero-copy host builder only marginally helped.
- Thread/process prefetch/pipeline did not engage usefully in _155/_155b.
- Python RAM-prefix lookup was catastrophic (~0.99M/min, high RAM/swap pressure).
- GPU prefix-3 prefilter worked but was saturated: ~92.6% positive rate, only ~7.4% skip.
- GPU prefix-4 prefilter was pathological on RTX 3050 6 GB: nearly full VRAM, 0% GPU util, very long runtime.
- 1M batch cap is unstable on this setup.
- 750k has historically caused OOM/downshift in longer tests.
- 700k looked slightly faster in 2 chunks but failed the 10-chunk stability run with OOM/downshift events; final learned safe batch was 500k, though 250k was reached temporarily during recovery.
```

## Current best settings to preserve

Current best scalar settings:

```bash
--v158-cuda-scalar-threads-per-block 128 \
--v158-cuda-scalar-geometry-label tpb128 \
--v159-fixed-window-bits 11 \
--v159-fixed-window-label win11
```

Current best lookup mode:

```bash
--hash160-lookup-mode mmap-bin-batch-fast20
```

Current batch-size status:

```text
500k = conservative stable
625k = current speed candidate, needs longer validation
700k = not production-safe yet; 10-chunk stability failed with OOM/downshift and learned back to 500k
750k/1M = not production-safe on current 16 GB / RTX 3050 setup
```

Practical current candidate command settings:

```bash
--cuda-vram-safe-batch-seeds 625000 \
--cuda-vram-min-batch-seeds 250000 \
--chunk-run-recovery-probe-up-max-seeds 625000
```

For conservative overnight stability, use 500k instead of 625k.

## Recent benchmark stats: fixed-window tuning

These are the important recent results. All were full `fast-text-core` scope, TPB128, 500k unless noted, mmap lookup, no OOM, memory OK.

| Release/test | Comparison | Throughput | Scalar GPU kernel total | Decision |
|---|---:|---:|---:|
| `_159` | win4 -> win5 | win4 ~24.17M/min, win5 ~25.97M/min | win4 ~96.64s, win5 ~80.26s | Promote win5 |
| `_160` | win5 -> win6 | win5 ~25.94M/min, win6 ~27.36M/min | win5 ~80.58s, win6 ~68.47s | Promote win6 |
| `_161b` | win6 -> win7 | win6 ~27.20M/min, win7 ~28.25M/min | win6 ~68.68s, win7 ~60.72s | Promote win7 cautiously |
| `_162b` | win7 -> win8 | win7 ~28.23M/min, win8 ~28.86M/min | win7 ~60.80s, win8 ~54.53s | Promote win8 cautiously |
| `_163` | win8 -> win9 | win8 ~29.28M/min, win9 ~29.83M/min | win8 ~54.51s, win9 ~51.06s | Promote win9 cautiously |
| `_164b` | win9 -> win10 | win9 ~29.75M/min, win10 ~30.03M/min | win9 ~51.03s, win10 ~47.47s | Promote win10 |
| `_165` | win10 -> win11 | win10 ~30.14M/min, win11 ~30.44M/min | win10 ~47.51s, win11 ~45.22s | Promote win11; stop window expansion for now |

Batch-size tests after win11:

| Release/test | Comparison | Throughput | Result |
|---|---:|---:|---|
| `_166` | 500k vs 625k | 500k ~30.61M/min, 625k ~31.03M/min | 625k +1.37%, clean 2 chunks |
| `_167` | 625k vs 700k | 625k ~31.02M/min, 700k ~31.15M/min | 700k +0.4%, clean 2 chunks |
| `_167` 10-chunk | 700k stability | OOM/downshift; final learned safe batch 500k | 700k not production-safe |

## Current decision state

The fixed-window scalar line produced real gains but has diminishing returns. Do not keep blindly testing win12/win13 unless there is a strong reason. The current best scalar setting is win11.

The next productive direction should likely be one of:

```text
1. Validate 625k + win11 over a longer fresh-root run, e.g. 10 chunks, because 700k failed and 625k is the best plausible middle-rung.
2. Route/lookup architecture work that avoids the failed RAM-prefix and failed GPU prefix-4 paths.
3. Exact lookup-stage optimization for current mmap-bin-batch-fast20: reduce Python/native wrapper overhead, improve typed route aggregation, and profile exact lookup per address family.
4. Deeper CUDA scalar/pubkey kernel work beyond fixed-window bits, e.g. table layout, register pressure, memory/coalescing, or a C++/CUDA feasibility path.
5. Later, add explicit Taproot/witness_v1_32 sidecar for audit coverage, but keep it separate from speed benchmarks.
```

## Immediate next task for the new chat

Start by asking me to upload or confirm the latest files. Then proceed with this order:

```text
A. If I have not uploaded them yet, ask for:
   - latest release script
   - latest result ZIP
   - latest README or handoff notes if available

B. Prepare a fresh-root 625k + win11 10-chunk stability script with one upload-ready ZIP.

C. If 625k is clean:
   - lock 625k as the current benchmark ceiling
   - compare against the 500k conservative baseline if needed
   - move to lookup/route-level optimization

D. If 625k fails:
   - lock 500k as production-stable
   - move to lookup/route-level optimization
   - do not keep pushing batch size upward
```

The first benchmark command in the new chat should preferably be:

```text
10 chunks
625k ceiling
TPB128
win11
mmap-bin-batch-fast20
full fast-text-core 10 transforms
fresh isolated scan root
one upload-ready ZIP
```

## Standard benchmark settings to reuse

Use these core options unless explicitly testing another variable:

```bash
--run-seedbin-chunk-range \
--allow-new-scan-root \
--chunk-run-max-chunks 10 \
--chunk-run-min-total-complete 0 \
--chunk-run-learn-safe-batch \
--chunk-run-min-learned-safe-batch-seeds 250000 \
--chunk-run-safe-batch-probe-after-clean-chunks 0 \
--chunk-run-safe-batch-probe-step-seeds 250000 \
--chunk-run-recovery-probe-up-after-clean-chunks 1 \
--chunk-run-recovery-cooldown-after-oom-chunks 1 \
--chunk-run-recovery-ignore-throughput-veto \
--chunk-run-throughput-governor \
--chunk-run-throughput-veto-min-base-candidates-per-min 11000000 \
--chunk-run-throughput-veto-memory-warning \
--chunk-run-soft-oom-governor \
--chunk-run-soft-oom-keep-min-base-candidates-per-min 11000000 \
--chunk-run-soft-oom-require-memory-ok \
--chunk-run-cuda-prewarm-seeds 4096 \
--debug-report-bundle \
--debug-report-zip \
--debug-report-copy-log-root \
--disable-native-host-dedup-builder \
--host-builder-mode fused \
--host-builder-omp-threads 8 \
--cuda-recursive-hybrid-route \
--auto-resume-batches \
--backend cuda \
--experimental-cuda \
--network mainnet \
--query-mode local \
--local-funded-db ./managed_addresses.sqlite \
--typed-address-lookup-db ./managed_addresses_typed_lookup.sqlite \
--typed-bin-dir ./managed_typed_bin \
--hash160-lookup-mode mmap-bin-batch-fast20 \
--recursive-batch-seeds 1250000 \
--cuda-vram-min-batch-seeds 250000 \
--cuda-vram-oom-retries 4 \
--v158-cuda-scalar-threads-per-block 128 \
--v158-cuda-scalar-geometry-label tpb128 \
--v159-fixed-window-bits 11 \
--v159-fixed-window-label win11 \
--derivation-suite fast-text-core \
--second-encoding none \
--third-encoding none \
--allow-partial-address-coverage \
--typed-include-address-types p2pkh_legacy,p2sh,p2wpkh \
--exclude-taproot-fallback \
--exclude-uncompressed-p2pkh-fallback \
--test-profile p2-cuda-full \
--production-fast \
--artifact-mode compact \
--progress-every 10000 \
--no-json \
--no-txt
```

For 625k validation, add:

```bash
--cuda-vram-safe-batch-seeds 625000 \
--chunk-run-recovery-probe-up-max-seeds 625000
```

For conservative 500k validation, use:

```bash
--cuda-vram-safe-batch-seeds 500000 \
--chunk-run-recovery-probe-up-max-seeds 500000
```

## How to analyze uploads

When I upload result ZIPs, extract and read:

```text
runner_summary.json
chunk_metrics.csv
batch_metrics.csv
outer_bash_log.txt
chunks/*/performance_statistics.txt
chunks/*/auto_resume_batches.jsonl
learned_safe_batch.json if present
bottleneck_report.md/json if present
```

Judge by:

```text
real/base candidates per minute
OOM events
memory pressure warnings
downshift behavior
learned safe batch
stage totals: scalar/pubkey, HASH160 lookup, host recursive build, GPU HASH160
whether experimental flags persisted correctly in metrics
```

Never claim speed improved unless benchmark reports show it.

## Next development priorities after 625k validation

If 625k validates:

```text
1. Lock 625k/win11/TPB128/mmap-fast20 as current benchmark ceiling.
2. Design lookup/route-level optimization release.
3. Profile exact lookup by address family: p2pkh_legacy, p2sh, p2wpkh.
4. Optimize mmap-bin-batch-fast20 wrapper/aggregation overhead.
5. Consider C++/CUDA experiments only for hot loops, not Python orchestration.
```

If 625k fails:

```text
1. Lock 500k as production-stable.
2. Stop batch-size escalation.
3. Move to lookup/route-level optimization and scalar kernel architecture.
```

Do not resume the mistake of treating reduced transform count as a serious fix. The software must support heavy full-scope tests efficiently.
