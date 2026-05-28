# MnemoAudit v92 Milestone

MnemoAudit is a sandbox/audit research tool for testing weak-brainwallet derivation hypotheses against managed Bitcoin mainnet address databases generated specifically for the research. It is designed to make the tested route scope explicit and auditable: a no-hit result is only meaningful for the derivation modes and address routes enabled in that run.

## Project scope

The tool reads a controlled source seed stream, applies selected derivation modes, derives private scalar candidates using the script convention `private_scalar = SHA256(converted_value_as_text_bytes)`, derives implemented Bitcoin address routes, and checks those routes against a typed managed-address lookup database.

It is not a wallet, not a malware framework, and not a tool for attacking third-party systems. It is intended for offline/sandboxed research against managed datasets owned or generated for the experiment.

## Important route profiles

### `--test-profile p2-cuda-fast`

Speed-oriented compressed P2 profile:

- compressed legacy P2PKH
- P2WPKH
- P2SH-P2WPKH
- excludes uncompressed legacy P2PKH
- excludes slow CPU uncompressed fallback

Use this when you want the fastest compressed-only P2 scan.

### `--test-profile p2-cuda-full`

Audit/full-P2 profile:

- compressed legacy P2PKH
- P2WPKH
- P2SH-P2WPKH
- CUDA/native uncompressed legacy P2PKH
- excludes slow CPU uncompressed fallback

Use this when uncompressed P2PKH coverage matters, including controlled calibration hits.

## Calibration guard

v92 adds clearer known-hit arguments:

```bash
--known-hit-seed "SEED"
--known-hit-address "ADDRESS"
--known-hit-route p2pkh_uncompressed
```

Supported routes:

- `p2pkh_compressed`
- `p2pkh_uncompressed`
- `p2wpkh`
- `p2sh_p2wpkh`

The known-hit guard validates that the expected seed/address/route is derivable under the MnemoAudit scalar convention before a large run.

## Recommended commands

Compressed-only production scan:

```bash
python3 mnemoaudit_btc_v26_cuda_92_milestone_audit_rc.py \
  --cuda-recursive-hybrid-route \
  --auto-resume-batches \
  --backend cuda \
  --experimental-cuda \
  --network mainnet \
  --query-mode local \
  --local-funded-db ./managed_addresses.sqlite \
  --typed-address-lookup-db ./managed_addresses_typed_lookup.sqlite \
  --typed-bin-dir ./managed_typed_bin \
  --test-profile p2-cuda-fast \
  --hash160-lookup-mode mmap-bin-batch-fast20 \
  --recursive-batch-seeds 1250000 \
  --auto-resume-cooldown-seconds 0 \
  --only-conversions sha256_encode,base64_encode,hex_encode,reverse,uppercase,lowercase \
  --second-encoding none \
  --production-fast \
  --artifact-mode compact \
  --progress-every 10000 \
  --no-json \
  --no-txt \
  -o ./out_p2_fast_92
```

Full P2 audit scan including CUDA/native uncompressed P2PKH:

```bash
python3 mnemoaudit_btc_v26_cuda_92_milestone_audit_rc.py \
  --cuda-recursive-hybrid-route \
  --auto-resume-batches \
  --backend cuda \
  --experimental-cuda \
  --network mainnet \
  --query-mode local \
  --local-funded-db ./managed_addresses.sqlite \
  --typed-address-lookup-db ./managed_addresses_typed_lookup.sqlite \
  --typed-bin-dir ./managed_typed_bin \
  --test-profile p2-cuda-full \
  --hash160-lookup-mode mmap-bin-batch-fast20 \
  --recursive-batch-seeds 1250000 \
  --auto-resume-cooldown-seconds 0 \
  --only-conversions sha256_encode,base64_encode,hex_encode,reverse,uppercase,lowercase \
  --second-encoding none \
  --production-fast \
  --artifact-mode compact \
  --progress-every 10000 \
  --no-json \
  --no-txt \
  -o ./out_p2_full_92
```

## Output scope reporting

`performance_statistics.txt` now includes a MnemoAudit route-scope summary indicating whether compressed P2PKH, uncompressed P2PKH via CUDA, uncompressed P2PKH via CPU fallback, P2WPKH, and P2SH-P2WPKH were enabled.

