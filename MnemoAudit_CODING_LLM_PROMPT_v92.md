# MnemoAudit coding prompt v92

Build an offline/sandbox audit tool named MnemoAudit for testing weak-brainwallet derivation hypotheses against managed Bitcoin mainnet address datasets generated for research. The tool must be explicit about scientific scope: a no-hit result is valid only for the selected derivation modes and selected address routes.

Core requirements:

1. Preserve deterministic source ordering and durable auto-resume markers.
2. Use typed address lookup families and report the enabled route scope in every performance/statistics artifact.
3. Implement two route profiles:
   - `p2-cuda-fast`: compressed P2PKH, P2WPKH, P2SH-P2WPKH only; excludes uncompressed P2PKH.
   - `p2-cuda-full`: p2-cuda-fast plus CUDA/native uncompressed legacy P2PKH; CPU uncompressed fallback remains disabled unless explicitly requested.
4. Preserve the scalar convention: `private_scalar = SHA256(converted_value_as_text_bytes)`.
5. Keep hit logging durable and append-only where possible.
6. Add known-hit calibration guards: seed, address, and expected route.
7. Never use probabilistic matching for final hits. Bloom filters or prefixes may only be prefilters if every potential hit is confirmed by exact full-payload comparison.
8. Clearly distinguish compressed and uncompressed P2PKH in code comments, logs, documentation, and performance files.
9. Include comments around native C/CUDA helpers explaining what is exact, what is a performance optimization, and what does not change research semantics.

Naming requirements:

- Use “MnemoAudit” in filenames, comments, documentation, CLI help, and user-facing logs.
- Avoid legacy project names in new output unless needed for backward compatibility.

Safety and audit wording:

The project is not a wallet, not a malware tool, and not an illicit key-search tool. It is an offline research/audit script for controlled managed datasets created specifically for the experiment.
