# MrHodlX

Bitcoin tooling and security engineering. I build offline, air-gapped wallet
software and test it like it owes me money.

## EntropyLab

Creator of [EntropyLab](https://github.com/MrHodlX/entropylab) — a
self-contained, single-file Bitcoin key and wallet calculator designed to run
fully air-gapped. The entire app is one HTML file with no network egress.

- **Never invent entropy:** all key material comes from user-supplied physical
  entropy (dice, coin flips, cards) or existing seeds — with live chi-squared
  fairness analysis that flags biased dice as you enter them.
- **Tested like a consensus implementation:** 157 test suites covering BIP32 /
  39 / 44 / 49 / 84 / 85 / 86 / 88 / 321 / 352 / 388 derivation, descriptors,
  PSBT handling, and aezeed — differential fuzzing against independent
  reference implementations (including a C++ LifeHash reference), plus an
  adversarial scenario harness.
- **Reproducible builds as a tested property:** the build runs twice from
  different staging paths and must produce byte-identical output; CI
  additionally requires the runner and the pinned dev container to agree on
  the digest.
- **Rust/WASM for the hot paths** — hdkey derivation, PSBT parsing, vanity
  search — with explicit stack scrubbing for secret-bearing buffers.

## Writing

- [Byte-identical builds as a test, not a promise](writing/byte-identical-builds-as-a-test.md)
  — how EntropyLab's CI makes reproducibility a property that fails the build
  instead of a claim in a README.

## Focus

- Bitcoin protocol security: derivation, parsing, PSBTs, script policies
- Adversarial and differential testing of cryptographic implementations
- Reproducible build verification and build-integrity tooling
- Privacy-preserving, network-silent software design

## Contact

The best way to reach me is an issue or discussion on
[entropylab](https://github.com/MrHodlX/entropylab). Open to Bitcoin security
reviews and protocol testing work.
