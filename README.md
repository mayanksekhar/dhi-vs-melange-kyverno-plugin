# Kyverno plugin image: Docker Hardened Images recipe vs a melange/apko build

A small, reproducible comparison from my work on docker-hardened-images/catalog
issue #302 (Kyverno Policy Reporter plugin images). It looks at one question:
what changes when the same upstream plugin is built through two toolchains?

## What I did

1. Wrote a DHI catalog definition for kyverno-plugin 0.6.1 (debian 13, Go
   build, hermetic, nonroot 65532). This is catalog PR #462.
2. Built the same upstream tag with melange and apko on the Wolfi package set,
   using a short recipe (git checkout pinned to the commit, then go/build).
3. Extracted the plugin binary and scanned it with trivy rootfs. I used rootfs
   on the binary because trivy image scanning reported 0 vulnerabilities for
   the same image, which was a false negative: Go modules in files owned by
   the OS layer were skipped.

## Results

| Build | Upstream version | HIGH or CRITICAL findings in the Go binary |
|---|---|---|
| melange/apko | 0.6.1 | 13 (across 7 Go modules) |
| melange/apko | 0.7.3 | 1 (golang.org/x/net v0.61.0, fixed in 0.62.0) |

Scanned with trivy 0.69.3 on 2026-11-22 (trivy rootfs, HIGH and CRITICAL only).
Two separate 0.6.1 builds gave the same 13 findings.

The one remaining finding in 0.7.3 (CVE-2026-78661) is also in the 0.6.1 list.
A naive attempt to bump that module on 0.7.3 failed: a transitive dependency
(supercronic v0.2.51) requires go 1.27.3 while the build requested go 1.27.0.
A bump needs a go directive change that I have not tested.

## What this does and does not show

- The big drop between 0.6.1 and 0.7.3 comes from upstream dependency updates,
  not from either toolchain. It says nothing about DHI versus Wolfi quality.
- Toolchain availability is the real difference I hit. Upstream moved to Go
  1.27 in kyverno-plugin 0.7.1 and trivy-plugin 0.5.1. At the time of writing
  the DHI catalog on main had 901 definitions using golang-1.26, 14 using
  golang-1.25 and none using golang-1.27, so the latest plugin versions could
  not be expressed as a DHI definition. The melange build used a newer Go
  from the Wolfi package set and built 0.7.3 without trouble.
- Not shown: byte for byte reproducibility of the compiled binary across
  rebuilds. I did not verify it, so I make no claim either way.
- Scan numbers are a point in time. Vulnerability databases move.

## Files

- melange.yaml: build recipe for 0.6.1
- melange-073.yaml: build recipe for 0.7.3
- apko.yaml: image assembly

## Reproduce the scan

Extract the built binary to a directory, then run:

    trivy rootfs --scanners vuln --severity HIGH,CRITICAL <directory>

## Build notes

apko.yaml refers to ./melange.rsa.pub. Generate your own signing key pair with
melange keygen, build the package with melange, then run apko against the
resulting local package repository.
