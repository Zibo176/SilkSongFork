# AnyPS5: Hollow Knight: Silksong (PPSA12544) fork

A fork of [boykopovar/AnyPS5](https://github.com/boykopovar/AnyPS5) for running **Hollow Knight: Silksong** (PPSA12544), a Unity / IL2CPP PS5 title. The `silksong` branch is the integration branch: it tracks upstream `main` and adds the changes below. Fixes are written to be cut from upstream `main` as PRs.

## Silksong status

Tested on an NVIDIA GeForce RTX 4060 Laptop GPU, Windows 11, October 2026.

| Part | State |
|------|-------|
| Boot: modules, Vulkan, swapchain | works |
| Native plugins (PSN, SaveData, Share, CommonDialog, Burst) | initialize with real args |
| IL2CPP runtime + managed assemblies | load and resolve |
| Shaders | compile + disk-cache |
| Intro / menu | renders and runs |

Silksong is the first Unity / IL2CPP title to run on AnyPS5: it boots fully, loads its Addressables bundles, initializes its native plugins, resolves the IL2CPP API, compiles and caches shaders, and renders the game.

Known issues: first-run shader compilation stutters until the on-disk cache warms; frame pacing and long-session stability are not yet measured.

## Changes on this branch

- **Deferred guest-module init (`--defer-sce-module`):** Unity native plugins are force-loaded as `sce_module` dependencies, so the generated Windows startup ran their init with zeroed argument registers and they failed a version handshake and crashed. `--defer-sce-module <file>` skips a module's startup init and the relinker instead emits an exported `__aps5_module_start` stub.
- **Run deferred init on `sceKernelLoadStartModule`:** when the game loads a deferred plugin, libkernel calls its `__aps5_module_start` with the real `(args, argp)`, so it initializes with correct arguments instead of the zeroed startup path.
- **Windows guest module/symbol resolution:** `dlopen` falls back to an already-resident guest module by name when a title requests it by its original package path (e.g. `/app0/Media/Modules/Il2CppUserAssemblies.prx`), and `dlsym` gains a Windows default-scope fallback (mirroring the Linux `RTLD_DEFAULT` path) so the title's runtime IL2CPP API lookups resolve.
- **Depth address reused as colour:** the AGC driver no longer rejects storage access to an address that was once a depth surface but is now reused as a plain colour/storage surface; `DepthPlaneView` checks the access is genuinely a live depth plane before refusing, so reused colour targets render instead of falling back to a stale snapshot.

## Running it yourself

You bring your own copy. This repository ships no game files, firmware, keys, or decryption tools: dump and decrypt your own legally-owned release, and extract its cleartext `eboot.elf` and bundled `sce_module/` modules yourself — that step is out of scope here. Tested on Windows 11 + RTX 4060 Laptop; Linux and macOS build the same way.

1. **Build the relinker and the system libraries** — follow [BUILD.md](docs/dev/BUILD.md) (the full build, not relinker-only: the fork's fixes live in the relinker *and* in libkernel/libSceAgcDriver, so you need the `libs` target too). On Windows, put the MinGW-w64 r7 `mingw64/bin` on `PATH` first. This yields `build/core/relinker/relinker.exe` and the patched `.prx` files in `build/core/libs/libs/`.

2. **Lay out your decrypted dump** so the relinker can read it:
   ```text
   source/
       eboot.elf
       sce_module/
           PSN.prx  SaveData.prx  Share.prx
           CommonDialog.prx  lib_burst_generated.prx
           ...  (plus the title's other bundled modules, e.g. the IL2CPP assemblies)
   ```

3. **Relink to a native executable, deferring the five native plugins.** This is the fork's whole point — deferred plugins initialize on the game's own `sceKernelLoadStartModule` call with real arguments, instead of crashing a version handshake on the zeroed-argument startup path:
   ```sh
   relinker --windows --registry \
       --defer-sce-module PSN.prx \
       --defer-sce-module SaveData.prx \
       --defer-sce-module Share.prx \
       --defer-sce-module CommonDialog.prx \
       --defer-sce-module lib_burst_generated.prx \
       source/eboot.elf outputs/app.exe
   ```
   Drop `--windows` for a Linux `app.elf`, or use `--macos` for a Mach-O (see [USAGE.md](docs/user/USAGE.md)).

4. **Assemble the runtime layout** beside `outputs/app.exe`, per [USAGE.md](docs/user/USAGE.md#runtime-layout): copy `build/core/libs/libs/*.prx` into `libs/` (on Windows also `libgcc_s_seh-1.dll`, `libstdc++-6.dll`, `libwinpthread-1.dll` from the toolchain), and copy the game's resources into `app0/` next to the `app0/sce_module/` the relinker just wrote.

5. **Run it** (Windows PowerShell, from `outputs/`, with the toolchain `mingw64/bin` on `PATH`):
   ```powershell
   $env:ANYPS5_GPU = "NVIDIA"   # optional: pick the Vulkan device by name
   .\app.exe
   ```
   SDL controllers and keyboard/mouse work out of the box; see [input mapping](docs/user/INPUT_MAPPING.md) for custom binds.

Expect the known limits from the [status table](#silksong-status): first-run shader compilation stutters until the on-disk cache warms, and frame pacing is not yet measured.

---

# About

Tool for automatic executables porting to Linux and Windows.

Includes a [relinker](core/relinker) that converts executable to the target system's native format and implementations of [system prx libraries](core/libs/prx) suitable for dynamic linking. No emulation or separate runtime process.

[Usage](docs/user/USAGE.md), [Build instructions](docs/dev/BUILD.md), [Architecture](docs/dev/ARCHITECTURE.md), [Technical debt of the project](docs/dev/TechnicalDebt.md), [code style conventions](docs/dev/CONVENTIONS.md), [contributing](CONTRIBUTING.md)

## Status

[![libraries](https://boykopovar.github.io/AnyPS5/badge-libraries.svg)](https://boykopovar.github.io/AnyPS5/) [![shaders](https://boykopovar.github.io/AnyPS5/badge-shaders.svg)](https://boykopovar.github.io/AnyPS5/)

[![progress map](https://boykopovar.github.io/AnyPS5/progress.svg)](https://boykopovar.github.io/AnyPS5/)

<sub>* System libraries: percentage of the functions known to the project so far (declared in [core/libs/prx](core/libs/prx)), not of every PS5 system function. The total grows as more functions are declared.</sub>

[List of verified games](docs/user/COMPATIBILITY.md)

Dreaming Sarah (2D platformer) runs at a stable 60 fps on a GTX 1050 Ti / i5-7500 3.4GHz.

Unsupported or unexpected states strictly throw `std::runtime_error`. `what()` is printed to stderr and the process terminates.

The [shader recompiler](core/shader/recompiler/Recompiler.cpp) successfully produces SPIR-V (validated via [Spirv-Tools](3rdparty/SPIRV-Tools) when built with `ANYPS5_ENABLE_SPIRV_TOOLS`).

## Compatibility

See the [game compatibility list](docs/user/COMPATIBILITY.md) for tested games and known issues.

## Input mapping

SDL-mapped game controllers are supported, including analog sticks and triggers. Keyboard and mouse controls can be configured with an `anyps5-input.ini` file. See [input mapping](docs/user/INPUT_MAPPING.md) for the supported devices and configuration format.

## Disclaimer

This project is intended for interoperability, research, preservation, and compatibility purposes. It does not include, distribute, or require copyrighted software, firmware, cryptographic keys, or proprietary libraries. Users are responsible for ensuring that any binaries used with this project are obtained and used in accordance with applicable laws and their respective license terms.

## License

This project is licensed under the GNU General Public License version 2 only.
