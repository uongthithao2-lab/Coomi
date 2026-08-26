# CoomiDev mobile build

This directory is the supported Linux ARM64 entry point for building the isolated CoomiDev APK inside Runtime V2.

## Runtime layout

The Android host persists `<engine-home>/runtime-v2/home/.coomi-dev` and mounts it in the Debian guest as `/opt/coomi-dev`:

```text
/opt/coomi-dev/
|-- bin/          Coomi-owned helper scripts
|-- current/      Selected, checksum-verified Build Kit
|-- toolchains/   Immutable Build Kit versions
|-- cache/        Gradle, Cargo and temporary build data
|-- state/        Installer state and manifests
|-- logs/         Build logs
`-- keys/         Local signing keys; never commit or print these
```

`current` must contain a `buildkit.json` matching `buildkit-manifest.schema.json`. Every native artifact must be a Linux AArch64/glibc executable or a platform-neutral JAR/script. Official Android SDK/NDK Linux host tools are commonly x86_64 and must not be treated as ARM64-compatible. Termux Android-PIE/Bionic executables under `/data/data/.../files/usr` must never be added to the guest PATH.

The repository does not bundle unverified third-party compilers. Install a separately obtained pinned archive only when its trusted SHA-256 is known:

```sh
coomidev-install-buildkit /path/to/coomidev-buildkit.tar.gz EXPECTED_SHA256
```

The installer verifies the archive before extraction, rejects unsafe/special entries, verifies every extracted file against `buildkit.json`, installs into a new immutable `toolchains/<id>` directory, and only then atomically selects it as `current`. It does not use `apt` or `dpkg`.

## Validation stages

Run the stages in order from a ProotLinux shell inside the repository:

```sh
coomidev-build doctor
coomidev-build android-smoke
coomidev-build rust-smoke
coomidev-build full
```

The full build sets `COOMI_DEV_BUILD=1`, limits Gradle/Cargo concurrency, uses the Build Kit's ARM64 `aapt2`, and exports the verified APK to `/home/coomi/CoomiDev-output`. Local build support is ready only after all four stages pass.

Use GitHub Actions as the supported fallback when an authenticated, checksum-verified ARM64 Build Kit is unavailable.

## GitHub Actions CoomiDev route (default)

The repository ships `.github/workflows/coomidev-apk.yml` for building the isolated
CoomiDev APK on a pinned `ubuntu-24.04` runner. It is the default route because
Android's official Linux host build tools are x86_64 while the phone guest is ARM64.

- Trigger manually: `gh workflow run coomidev-apk.yml --ref <branch> -f version_suffix=1.4.4-coomidev.1`
- Inputs: `ref` (defaults to the triggering branch), `version_suffix` (semver), `upload_apk`.
- The workflow runs `npm run type-check`, the smallest relevant Rust tests
  (`cargo test -p coomi-security --lib`), then `COOMI_DEV_BUILD=1 ./gradlew :app:assembleRelease`.
- Runtime V2 assets (`proot-host-arm64.tar.gz`, `debian-bookworm-arm64.tar.gz`) are
  downloaded from the pinned official release and verified against the committed
  `apps/coomi-app/app/src/main/assets/runtime-v2-manifest.json` SHA-256 entries.
- Preview signing uses the fork's encrypted secrets `COOMI_PREVIEW_KEYSTORE_B64`
  (base64 of a JKS, alias `alias`) and `COOMI_PREVIEW_KEYSTORE_PASSWORD`
  (store/key password). The workflow fails when the secrets are missing; it never
  falls back to plaintext or prints key material.
- After packaging, the workflow verifies package `com.coomidev.android`, label
  `CoomiDev`, ABI `arm64-v8a` and the APK signature, then uploads the
  `coomidev-apk` artifact with a SHA-256 checksum.

