# How KytyPS4 Runs PS4 Games on Android

> The full breakdown of the actual pipeline: import, runtime, launch, rendering, input.
> No hand-waving — these are the real code paths in this repository.

---

## The short version

PS4 games are x86_64 binaries. Android phones are ARM64. This project does **not** try to
interpret the game itself — instead it runs the actual open-source **shadPS4** emulator on
the phone, and uses **Box64** to translate shadPS4's x86_64 code to ARM64 at runtime.

So there are two layers doing two different jobs:

1. **Box64** — dynamic binary translation. It translates the x86_64 `shadps4` binary (the
   emulator) into ARM64 instructions so it can run on Android's CPU.
2. **shadPS4** — PS4 userland emulation. Once running, shadPS4 provides the PS4 (Orbis)
   system libraries to the game: libkernel, filesystem, graphics, etc.

Everything else — display, GPU drivers, input, and lifecycle — is Android-side plumbing.

---

## 1. Import — getting the game onto the phone

Games are distributed as `.pkg` files. The app uses the system document picker (SAF) to
open one, then the native `pkg` library unpacks it:

```
PKG file (.pkg)
    |
    | ImportService (foreground dataSync)
    |   + PkgRsa (Java)  -> RSA signature verification of the package
    |   + pkg_aes_arm    -> AES-128 key decrypt (ARMv8 crypto instructions)
    |   + pkg_extractor  -> 4 parallel worker threads
    v
files/games/<game>/
    +-- eboot.bin        (x86_64 Orbis ELF — the main executable)
    +-- *.prx            (PS4 shared libraries)
    +-- sce_sys/ ...     (metadata, icons, param.sfo)
```

Import is CPU-bound, so extraction runs across four parallel workers while a progress
notification stays in the notification drawer.

### Crypto

- Package keys are decrypted with AES-128. On ARMv8 devices the crypto extension is used
  (`pkg_aes_arm.h`), with a software AES fallback (tiny-AES-c) when the hardware path is
  unavailable. The path is verified with a NIST FIPS-197 test vector at startup and logged
  under `Kytyps4Pkg` (`"ARMv8 AES hw path enabled/disabled"`).
- RSA signatures are checked on the Java side (`PkgRsa`) as the fast path.

---

## 2. Runtime — Box64 + shadPS4

The emulator needs two things installed before a game can run:

- **Box64** — the x86_64 → ARM64 translator
- **shadPS4** — the PS4 emulator core (x86_64 Linux build)

The runtime is installed into `files/runtime/` (e.g. `box64-*` directories). In Play builds
it is bundled in the APK; other builds download `manifest.json` + `runtime.zip` from the
release repo and verify it.

### Guest patches

Android's seccomp policy (arm64_app) blocks syscalls like `statx()` and
`name_to_handle_at()`. The guest's `libudev` calls them via `statx@plt`, which would kill
the whole process with SIGSYS (exit 159). `patchGuestLibudev` swaps in a patched
`libudev.so.1.7.10` whose `statx` / `name_to_handle_at` PLT call sites redirect to stubs
that return `ENOSYS` (see `tools/patch_libudev.py`). The patch only applies when the
installed file matches the expected md5, so a different runtime version is left untouched.

### Launch line

`RuntimeProcessLauncher` builds the actual command (with path-safety validation so nothing
escapes app storage):

```
<host loader> --library-path <runtime host dir> <box64> <shadps4> \
    --override-root <game override> \
    --bachata-storage-root <storage> \
    --bachata-socket <control socket>
```

Box64 runs in one of two modes:

- **HOST_GLIBC** — a custom host loader + `libbox64.so` (host glibc line)
- **APK_NATIVE** — the bionic `libbox64.so` shipped in the APK

---

## 3. Display — Winlator-style X server

PS4 Linux games expect a graphical environment. The guest runs under an embedded X server
(`WinlatorEmbeddedXServer`) whose output is rendered onto the `SurfaceView` that the
session screen owns. The surface is attached to the runtime (`ManagedSession.attachSurface`)
before the guest boots so the display is ready when the game starts rendering.

---

## 4. GPU — Mesa Turnip + Vortek

shadPS4 renders through Vulkan. On Android:

- **Mesa Turnip** provides the Vulkan 1.1 driver for Adreno GPUs. Bundled driver lines are
  `mojo-26.1`, `mojo-25.0`, `gen8`, and `pojav`; the driver directory and library are
  selected per device and exported to the guest via `BACHATA_VULKAN_DRIVER_DIR` /
  `BACHATA_VULKAN_DRIVER_NAME`.
- **Vortek** (experimental) provides the Vulkan WSI/translation layer — the session starts
  the Vortek server over a local socket when the system-driver path is selected.

The BACHATA_* environment variables (`BACHATA_VORTEK_*`, `BACHATA_CRASH_REGISTERS`,
...) control the guest's GPU/ABI behaviour and are logged in
the session report for debugging.

---

## 5. Input and telemetry — the control socket

A local Unix socket (`runtime-control.sock`) carries a small line protocol between the app
and the guest (`BACHATA/1`):

- `BACHATA/1 INPUT slot=<n> ...` — gamepad state frames
- `BACHATA/1 EVENT Running|Frame` — guest lifecycle/rendering events
- `BACHATA/1 ERROR code=...` — guest errors

Input comes from a physical gamepad (`GamepadInputManager`) or the on-screen touch layout
(`FixedControllerOverlay`, per-game layouts), is encoded into controller snapshots, and is
forwarded to the guest. FPS / frame-time telemetry is measured from `EVENT Frame` and shown
on the HUD overlay.

---

## 6. Session lifecycle and diagnostics

`EmulationService` is a foreground service. Each session:

1. creates a `SessionLog` (`files/logs/<timestamp>-<game>-<suffix>/`) with `application.log`
   and `shadps4.log`
2. validates the game root, installs/patches the runtime, waits for the surface
3. launches the guest, mirrors logs live into the session log
4. on stop/crash/failure, copies the backend log and exports the session directory to
   external `files/logs/`, pruning old sessions

Uncaught crashes are recorded by `CrashLog`, surfaced on the next launch by `CrashScreen`,
and the app also ships a `CrashActivity` that runs in its own `:crash` process so a broken
main process can still show the report. Reports and session archives can be shared from the
diagnostics screens.

---

## Performance notes

| Layer | Cost | Notes |
|-------|------|-------|
| Box64 DBT | ~2-4x | Only translates the emulator, not per-frame game code |
| shadPS4 emulation | game code runs native | PS4 libraries are HLE'd, GPU is Vulkan natively |
| AES decrypt (import) | negligible | ARMv8 crypto extension hardware path |
| Driver choice | per-SoC | mojo vs gen8 vs pojav picked by device |
| Real-world | good | **Sonic Mania: 45–50 FPS on Snapdragon 855** (Mesa Turnip) LSFG frame Generation | 

The main compatibility lever is the per-game profile: driver line, Box64 env, and runtime
settings are all stored per title and applied at launch.
