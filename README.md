<p align="center">
    <a href="https://github.com/dev-Ali2008/KytyPS4-Android/blob/38dc3897c9bf4e1fa0f38811ff681b921832fb05/kytyps4-logo.png">
        <img height="150px" src="https://github.com/dev-Ali2008/KytyPS4-Android/blob/38dc3897c9bf4e1fa0f38811ff681b921832fb05/kytyps4-logo.png" />
    </a>
</p>

# KytyPS4 — PS4 Emulator for Android

A PS4 emulator for Android built on [shadPS4](https://github.com/shadps4-emu/shadps4)
running under [Box64](https://github.com/ptitSeb/box64), with a Winlator-style embedded X
server, Vulkan through **Vortek** and **Mesa Turnip** drivers. Kotlin/Compose UI talking
to the C++ core and guest runtime through JNI.

Fork From — [Bachata S4](https://github.com/JICA98/Bachata-S4)

ourselves, with special attention to making it work on **Android 10**

>This is a real emulator: it runs actual PS4 games on your phone Android 10-11 

> the will be ps5 in emu but soon 

## Repository Policy

**Repository Purpose:** To safeguard the codebase and maintain proprietary performance enhancements (KytyPS4 Android is closed-source This official) repository does not contain the emulator's source code Instead, it serves as the official version archive and primary distribution platform for releasing APK builds managing compatibility issues and sharing user guides...

## Status

From my testing in games like Undertale + sonic mania. It boots and reaches gameplay at roughly 40 to 50 fps on a Snapdragon 855 6 Ram + LSFG Frame Generation 
— [How Its work?](https://github.com/dev-Ali2008/KytyPS4-Android/blob/main/How_ITS_work_.md)


### Screenshot - Test
<table align="center">
  <tr>
    <td align="center">
      <strong>Xeodrifter</strong><br>
      <img src="https://github.com/dev-Ali2008/KytyPS4-Android/blob/a8ddb6933c00d1a3cb10d215f553bc26ceaf4dc5/Screenshot-Test/Screenshot-1.png" width="300" alt="Xeodrifter running in KytyPS4">
    </td>
    <td align="center can">
      <strong>Undertale</strong><br>
      <img src="https://github.com/dev-Ali2008/KytyPS4-Android/blob/a8ddb6933c00d1a3cb10d215f553bc26ceaf4dc5/Screenshot-Test/Screenshot-2.png" width="300" alt="Undertale running in KytyPS4">
    </td>
  </tr>
  <tr>
    <td align="center">
      <strong>Balatro ( just start And then crash ) </strong><br>
      <img src="https://github.com/dev-Ali2008/KytyPS4-Android/blob/a8ddb6933c00d1a3cb10d215f553bc26ceaf4dc5/Screenshot-Test/Screenshot-3.png" width="300" alt="Balatro running in KytyPS4">
    </td>
    <td align="center">
      <strong>Sonic Mania</strong><br>
      <img src="https://github.com/dev-Ali2008/KytyPS4-Android/blob/a8ddb6933c00d1a3cb10d215f553bc26ceaf4dc5/Screenshot-Test/Screenshot-4.png" width="300" alt="Sonic mania running in KytyPS4">
    </td>
  </tr>
  <tr>
    <td align="center">
      <strong>silksong</strong><br>
      <img src="https://github.com/dev-Ali2008/KytyPS4-Android/blob/a8ddb6933c00d1a3cb10d215f553bc26ceaf4dc5/Screenshot-Test/Screenshot-5.png" width="300" alt="silksong running in KytyPS4">
    </td>
    <td align="center">
      <strong>Dead.Cells</strong><br>
      <img src="https://github.com/dev-Ali2008/KytyPS4-Android/blob/a8ddb6933c00d1a3cb10d215f553bc26ceaf4dc5/Screenshot-Test/Screenshot-6.png" width="300" alt="Dead cells running in KytyPS4">
    </td>
  </tr>
</table>

<p align="center"><em>And many more...</em></p>


## Tech Stack

| Component | Role |
|-----------|------|
| [Bachata S4](https://github.com/JICA98/Bachata-S4) | Used as a reference/help only |
| [shadPS4](https://github.com/shadps4-emu/shadps4) | PS4 (Orbis) userland emulation core |
| [Box64](https://github.com/ptitSeb/box64) | x86_64 → ARM64 dynamic translation for the guest emulator |
| Winlator | Embedded X server for guest display |
| [Vortek](https://github.com/KhronosGroup/Vortek) | Vulkan 1.1 translation / WSI for the guest |
| [Mesa Turnip](https://gitlab.freedesktop.org/mesa/mesa) | Vulkan 1.1 driver for Adreno |
| Kotlin + Jetpack Compose | UI (library, settings, drivers, session, diagnostics) |
| Native C++ (CMake / NDK) | PKG crypto & extraction, Vortek bridge, custom Vulkan bridge |

## Architecture

```
Kotlin UI (Compose)
    |
    | JNI (pkg, vortek_server, custom_vulkan_bridge)
    v
EmulationService (foreground)
    |-- RuntimeProcessLauncher
    |       `-- host loader -- library-path -- box64 -> guest shadPS4 (x86_64)
    |-- WinlatorEmbeddedXServer  (display -> SurfaceView)
    |-- VortekSessionSupport     (Vulkan WSI)
    |-- VulkanDriverConfiguration (BACHATA_VULKAN_DRIVER_* -> Mesa Turnip)
    |-- control socket (BACHATA/1: frame telemetry, input)
    `-- SessionLog / CrashLog    (diagnostics)
    |
    v
ImportService  --native pkg lib-->  game root (files/games/... eboot.bin)
```

## Credits

- [Bachata S4](https://github.com/JICA98/Bachata-S4) — used as a reference/help during development
- [shadPS4](https://github.com/shadps4-emu/shadps4) — PS4 emulation core
- Winlator — embedded X server
- [Box64](https://github.com/ptitSeb/box64) — x86_64 translation
- [Vortek](https://github.com/KhronosGroup/Vortek) — Vulkan translation
- Mesa Turnip drivers

## License

See `LICENSE.txt`. The project builds on the licenses of Bachata S4 - shadPS4 - Winlator - kytyps5

## Disclaimer

This software is for educational and research purposes only. It is not affiliated with or
endorsed by Sony Interactive Entertainment. PlayStation is a registered trademark of Sony
Interactive Entertainment.
