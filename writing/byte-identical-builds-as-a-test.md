# Byte-identical builds as a test, not a promise

*How EntropyLab makes "reproducible build" a mechanical property that fails
CI, instead of a claim in a README.*

Most Bitcoin wallet projects claim reproducible builds. Almost none of them
can tell you the moment the claim breaks — because nothing in the pipeline
actually checks it. EntropyLab's whole pitch is "audit this one HTML file
offline," so a build that isn't reproducible isn't a nuisance, it's a broken
security promise. Here's how we test it instead of asserting it.

## The threat

EntropyLab ships as a single self-contained `entropylab.html`. The user
verifies a published sha256, downloads the file, and air-gaps it. For that
checksum to mean anything, anyone must be able to rebuild the same bytes from
the sources. If the build silently depends on the machine, the path, or the
time of day, then "reproducible" is marketing, and the signed sums are
unverifiable in practice.

The classic leaks: absolute paths baked into output, timestamps, locale- or
timezone-dependent formatting, filesystem ordering, and environment variables
reaching the artifact.

## Layer one: same runner, two hostile path shapes

`scripts/reproduce.mjs` builds the working tree twice in one run — once
shallow, once under a deliberately deep nested path:

```
/tmp/entropylab-repro-XXX/a
/tmp/entropylab-repro-XXX/nested/deeper/b
```

Both stages are *copies* of the working tree — not `git archive` — so the
check covers the tree as it stands, committed or not, and carries no `.git`.
Then it sha256s both `entropylab.html` and `service-worker.js` and requires
identical digests, or exits 1.

The two path shapes are the point: a leaked absolute path, or any
length-dependent layout behavior, shows up as a digest mismatch between two
builds run seconds apart on the same machine. No "works on my machine" escape
hatch.

This runs as an ordinary test in the default suite
(`test/build-reproducible.test.mjs`), so every `npm test` re-proves it.

## Layer two: runner vs. pinned container, byte for byte

Same-machine agreement isn't enough — esbuild and friends can differ across
environments. The CI `reproduce` job therefore:

1. Builds on the GitHub runner and records `sha256sum entropylab.html service-worker.js`.
2. Builds the pinned dev image (the `Dockerfile` is the canonical build
   environment: pinned Node, pinned Rust wasm toolchain).
3. Builds *again inside that image* on a mounted checkout and `diff`s the two
   sum files.

The release pipeline `needs` this job. A digest mismatch between the runner
and the image doesn't just fail CI — it stops the Pages deploy and artifact
publication. The claim is enforced exactly where the artifact ships.

## The subtle one: don't let the fix invalidate itself

First pass at this had a trap: the footer stamps the commit that produced the
build. Stamp `HEAD` and the very commit that adds release signatures changes
the bytes — invalidating the sums you just signed. A chicken-and-egg bug that
makes reproducibility useless at release time.

`scripts/build-commit.mjs` instead stamps **the most recent commit that
changed a build input** (`src/`, the build scripts, the pinned lockfile,
manifest — an explicit allowlist). Docs, changelog, and signature commits
leave `entropylab.html` byte-identical, so the sums signed for a release stay
valid when that commit lands. Reproducibility that only holds between
releases is a demo; this is what makes it hold *across* the release process.

Two more mechanical guards, because deterministic-by-accident isn't
deterministic:

- The service-worker cache version is a **hash of the output**, not a
  timestamp — same inputs, same version string.
- After token replacement, the build greps its own output for unreplaced
  tokens and HTML-forbidden control characters and throws. A dependency that
  starts emitting one fails the build instead of shipping a weird diff.

## What this catches that a promise doesn't

- A new dependency that embeds `__dirname` or `import.meta.url` paths →
  layer one mismatch.
- Toolchain drift between dev machines and CI → layer two mismatch, deploy
  blocked.
- "Just stamp HEAD, it's simpler" refactors → release sums break, loudly.
- Nondeterministic ordering in a bundler update → mismatch on the next
  `npm ci`.

## The takeaway

Reproducibility isn't a property of your intentions; it's a property of your
pipeline. The cheap version — two builds, two weird paths, compare hashes —
takes an afternoon to add to any project and catches the majority of real
leaks. The full version — runner vs. pinned image gating the deploy — is what
turns "we believe it's reproducible" into "it cannot ship unreproducible."

If your wallet project signs checksums but nothing in CI would fail when they
stop matching, you have a marketing page, not a reproducible build.

---

*Source: [`scripts/reproduce.mjs`](https://github.com/MrHodlX/entropylab/blob/main/scripts/reproduce.mjs),
[`scripts/build-commit.mjs`](https://github.com/MrHodlX/entropylab/blob/main/scripts/build-commit.mjs),
and the `reproduce` job in
[`ci-cd.yml`](https://github.com/MrHodlX/entropylab/blob/main/.github/workflows/ci-cd.yml).*
