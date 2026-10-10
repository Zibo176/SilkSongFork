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
