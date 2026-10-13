# Harmony's Odyssey (PPSA16827) — AnyPS5 Compatibility Report

## Summary

This branch documents an AnyPS5 Windows test run for **Harmony's Odyssey**
(Title ID: `PPSA16827`, content version `01.000.000`).

- Engine: Unity 2021.3.25f1 (IL2CPP)
- Host: Windows x86-64
- Target output: Windows PE (`HARMONYPS5.exe`)
- GPU/runtime: AGC/Vulkan path reached during the launch test

## Important compatibility-classification caution

**The screenshot attached for this report comes from `harmony-odyssey-workaround`,
not from strictly unmodified current `main`.**

Clean current `main` does not reach a rendered first frame for this game because
`libSceAgcDriver.prx` fails to load with Windows error 127:

```text
Failed to load module: converted\libs\libSceVideoOut.prx
GetLastError: 127 - The specified procedure could not be found
```

Observed missing cross-library symbol:
```text
_ZN10LibcUnwind11CursorSlotsE
```

This branch contains a minimal experimental workaround for that issue:
- `core/libs/prx/libc/include/exceptions/Unwind.hpp`
- `core/libs/prx/libc/src/exception/Exports.cpp`

That change should be reviewed independently before use.

## Build/runtime results

- Relink completed successfully with `--windows --registry`.
- Decrypted ELF inputs were used.
- Game assets and module layouts were staged under `converted/app0`.
- Built AnyPS5 libraries: `build/core/libs/libs/*.prx`
- Runtime was stable for at least 60 seconds after the workaround.
- No `NOT IMPLEMENTED` / stub-entry messages were observed in the captured runs.
- No controller-driven feature-path run triggered a stub call.

### Import audit
- ELF dependency scope: 56 direct dependencies
- Total relocation references: 2607
- Unique imports: 1068
- Implemented: 900
- Stub-backed: 25
- Module-backed: 143
- Absent: 0

## Runtime warning / latent risk
The following stub-backed symbols exist but were **not** triggered in the observed windows. They remain feature-path risks:

- AGC capture/semaphore-related markers
- Browser dialog cookie APIs
- SSL name-entry APIs
- SystemService stubs
- Game update / addcont API
- NP trophy reward icon API
- `__tls_get_addr` and related compatibility stubs

## Visual evidence
Screenshots in:
- `screenshots/harmony-odyssey/runtime-01-workaround.png`
- `screenshots/harmony-odyssey/runtime-02-workaround.png`

Associated branch:
- `harmony-odyssey-workaround`

Screenshot status: these are **modified-branch screenshots**, not clean unmodified-main screenshots.

## Suggested roadmap wording
Practical wording for the issue form:

```text
Harmony's Odyssey (PPSA16827) reaches a rendered window and remains stable for 60+ seconds only after a minimal local workaround. Clean unmodified current main is blocked before the first frame due to a libSceAgcDriver/libc unwind-symbol load failure.
```

Recommended stage classification:

```text
No stage yet; claimed Menus, unverified
```

## Evidence references
- `logs/import_audit_harmony_2026-10-13.json`
- `converted/stdout.log`
- `converted/stderr.log`
- Branch: `harmony-odyssey-workaround`

## Note for recording the result
Use the appropriate user-facing stage and identify whether the screenshot is from a modified tree or unmodified `main`.
