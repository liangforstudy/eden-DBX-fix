<!--
# SPDX-FileCopyrightText: Copyright 2025 Eden Emulator Project
# SPDX-License-Identifier: GPL-3.0-or-later

# SPDX-FileCopyrightText: 2018 yuzu Emulator Project
# SPDX-License-Identifier: GPL-2.0-or-later
-->

# Eden-DBX-fix

An unofficial fork of the [Eden](https://git.eden-emu.dev/eden-emu/eden/) Nintendo Switch emulator. It fixes the Vulkan "Device loss" crashes in Dragon Quest X Offline (0100E2E0152E4000), reported upstream in [Issue-Reports #777](https://github.com/eden-emulator/Issue-Reports/issues/777). The crash fix is confirmed after hours of play. A separate problem with rooms turning bright after long sessions is still being investigated, see [Known issues](#known-issues).

(The "DBX" in the name is a typo for DQX.)

Everything else is upstream Eden as of 4 Nov 2026. For general documentation and support, go to the upstream project:
https://git.eden-emu.dev/eden-emu/eden/

Please don't report problems with this fork to the Eden developers. Open an issue here instead.

## Explainer

**[Eight Rows, Two-Row Texture](https://claude.ai/artifact/42GP1E9sRSaPBo64rVkBVv)** is an interactive page that shows the bug: the game copies an 8×8 texture into a texture Eden made only 8×2, and the extra rows land in GPU memory that doesn't belong to it. You can step through the copy before and after the fix.

If anything there (or here) is confusing, [open an issue](../../issues) and ask. No question is too basic.

## Status

| Cause | What it does | Status |
|---|---|---|
| **A.** Page-table block id overflow | Newer Eden master builds log `assert current_block != 65535`, then lose the GPU about 100 s later | Fixed |
| **B.** Oversized DMA (direct memory access) texture copy | Loses the GPU on a loading screen or map change, 2 to 25 minutes in. This is the crash in #777 | Fixed, confirmed |
| Bright interiors and noisy textures | After about 2 hours, some rooms turn blinding white until Eden restarts | Not fixed, being investigated (see [Known issues](#known-issues)) |

| Run | Machine | Build | Result |
|---|---|---|---|
| 1 | GTX 1050 | Cause A fixed | Device loss at 632 s |
| 2 | GTX 1050 | Same, with Vulkan validation | Device loss at 156 s. Validation caught three oversized copies 7 ms before it |
| 3 | GTX 1050 | Plus the upload clamp | No device loss in 13 minutes. 8 oversized copies caught, none reached the GPU |
| 4 | GTX 1050 | Plus the DMA size check | No device loss in 6 minutes. The bad copy didn't come up in this run |
| 5 | RTX 5060 | Test build 1 | No device loss in two sessions of about 3 h 45 min each. The size check caught the bad copies, and the clamp never had to step in |
| 6 | GTX 1050 | Test build 2 | No device loss in 11.5 minutes. The size check caught 11 bad copies |
| 7 | RTX 5060 | Test build 2 | No device loss and no bright room in 20 minutes |

The GTX 1050 is an i7-8750H laptop with driver 546.33. The RTX 5060 is the original i7-14650HX laptop from the bug report.

## Known issues

### Bright interiors and noisy textures

After about 2 hours of play, some house interiors become extremely bright: only the minimap and menus stay visible. Once a room breaks it stays broken, even after leaving, re-entering or loading an older save, until Eden is restarted. Any house can be affected, and rooms visited earlier keep looking normal. Some textures, such as floor tiles and character portraits, can also turn into streaky noise.

This isn't caused by the crash fix: it happened before it too, but the crash stopped anyone playing that long. Because a restart fixes it and loading a save doesn't, the bad data is most likely in Eden's texture cache, which keeps textures on the GPU and only throws old ones away when memory runs low. That would explain why it takes hours. The exact cause isn't known yet, and none of the shorter test runs so far have reproduced it.

Test build 2 has switches to help reproduce it faster. See [Test switches](#test-switches). If a room turns bright for you, please [open a Crash report](../../issues/new/choose) with which house it was, roughly when, and the log.

## Download

Get `eden.exe` from the [Releases](../../releases) page. Use the newest one, **test build 2** (`dqx-fix-2`): a Release build with the crash fix and the test switches below. The symbols zip is only needed for analysing a crash dump from that exact build.

1. Install the latest [Microsoft Visual C++ Redistributable (x64)](https://learn.microsoft.com/cpp/windows/latest-supported-vc-redist). The build uses the Visual Studio 2026 runtime, and with an older one installed `eden.exe` may not start.
2. Run `eden.exe`. Windows will show **"Windows protected your PC"**, because the file isn't signed. Click **More info**, then **Run anyway**. This only appears the first time.
3. It uses your normal `%APPDATA%\eden` folder, so your games, keys and settings carry over.

Building it yourself is possible (see [Building](#building)), but it takes a few hours of setup. That's why there's a ready-made exe.

If it still crashes, close Eden, then attach `%APPDATA%\eden\log\eden_log.txt` to an issue here. Copy the log before you launch Eden again, because each launch replaces it.

### Test switches

Test build 2 has two switches for hunting the bright-room bug. Both are off unless you set them, and they only apply when Eden is started from PowerShell like this, in the folder where you unzipped it:

    $env:EDEN_DQX_TEXCACHE_SCALE=10; $env:EDEN_DQX_TRACE=1; .\eden.exe

- `EDEN_DQX_TEXCACHE_SCALE=10` makes the texture cache throw old textures away ten times sooner, so the memory pressure of hours of play arrives within minutes. Expect some stutter.
- `EDEN_DQX_TRACE=1` logs every creation and deletion of small RGBA16F textures, the kind the bad copies target.

Turn graphics debugging off for these runs (Emulation > Configure > Debug), because it slows the game down a lot. A good test is to go in and out of every house in a town, plus a few town exits and loading screens, for 20 to 30 minutes.

To note the moment something goes wrong, paste this into a PowerShell window once, changing the path to a folder of your own:

    function mark($t) { $s = ((Get-Date) - (Get-Process eden | Select -First 1).StartTime).TotalSeconds; $line = "[{0,12:N6}] NOTE: {1}" -f $s, $t; Add-Content "$HOME\Desktop\eden_notes.txt" $line; $line }

Then type something like `mark "bright room, inn on the east side"`. Each note is stamped with Eden's log time, so it can be matched against the log.

## Background

This fork started with Dragon Quest X Offline (Japanese, v2.0.1 with the English patch) crashing on Eden v0.2.1, on an i7-14650HX laptop with an RTX 5060 Laptop GPU. About half the time, the game crashed on a loading screen after 6 to 24 minutes of play, and every log ended with:

    Render.Vulkan <Critical> vulkan_device.cpp:773:ReportLoss: Device loss occurred!

None of these stopped the crash:

- Disabling Extended Dynamic State, raising GPU accuracy to High, and lowering Fast GPU Time
- Updating the NVIDIA driver from 591.91 to 617.42, and clearing the shader cache
- Raising the Windows GPU timeout (TdrDelay and TdrDdiDelay set to 10 seconds)
- Removing the English patch, and removing the update that adds the DLC content
- Setting the laptop's MUX switch to dGPU-only mode, which rules out the dual-GPU laptop issue ([Issue-Reports #257](https://github.com/eden-emulator/Issue-Reports/issues/257))

It also reproduces on a second laptop with a GTX 1050. The same game files run on Ryubing without crashing, only slower, which pointed to Eden rather than the game or the hardware.

## What changed

### Oversized DMA texture copies (`src/video_core/texture_cache/texture_cache.h`)

The Switch GPU has a copy engine that games use to move pixel data into textures. It works by DMA (direct memory access): it copies memory on its own, without the CPU doing the work. Eden speeds it up by finding the texture it already has at the destination address and copying straight into it on the host GPU. The copy size comes from the game's command, and Eden never checked it against the texture it found.

In Dragon Quest X, the game copies an 8×8 RGBA16F block (plus 4×4 and 2×2 levels) to an address where Eden's texture is only 8×4, 8×2 or 8×1, with one level. The extra rows are written past the end of the texture. When that memory isn't mapped, the GPU faults and Windows resets the driver.

`DmaImageId` now checks that the copy's rectangle fits the texture level it found. If it doesn't, Eden does the copy through emulated memory instead, and the texture is rebuilt from there. The first 64 cases are logged in full, then every 64th with a running count:

    DMA upload #1 of 8x8 at (0,0) does not fit level 0 (8x4) of image 8x4, using the software path. Guest surface ... 64x8x1 ...; image id=117 ... levels=1 ...

Each line shows both what the game says the surface is and which texture Eden matched. In Dragon Quest X the game treats each address as a full 8×8 surface (64 bytes × 8 rows, 512 bytes apart), while Eden's textures there are smaller render targets the GPU drew into.

### Upload clamp (`src/video_core/renderer_vulkan/vk_texture_cache.cpp`)

As a safety net, any buffer-to-image upload that is bigger than its destination level is trimmed to fit, using the size the Vulkan image was created with. The first 32 cases are logged as `Oversized ... upload clamped`. With the DMA check in place, this shouldn't trigger.

### Memory span contiguity check (`src/core/memory.cpp`)

Every memory mapping gets a block id from a 16-bit counter that is never reset, so after 65,535 mappings unrelated mappings can share an id. `GetSpan` treated a range as contiguous if the first page and the page after the end had the same id, so it could return a pointer spanning two separate host regions. It's used for IPC buffers, audio and NVDEC command buffers.

The fix checks every page in the range for the same host pointer base and block id. Otherwise `GetSpan` returns `nullptr` and the caller uses the normal copy path. This also fixes an off-by-one on the end page. The `current_block` assert still appears in the log, but it's now harmless.

### Device loss location logging (`src/video_core/...`)

`Device loss occurred!` now says where the loss was detected: `(scheduler submit)`, `(present blit submit)` or `(query results)`.

### Bright-room diagnostics (`src/video_core/texture_cache/`)

Test-only tools for the [bright interiors](#known-issues), all off by default: the `EDEN_DQX_TEXCACHE_SCALE` and `EDEN_DQX_TRACE` [test switches](#test-switches), and a log line with the size, format and layout of any texture using `tile_width_spacing`, a layout Eden doesn't support yet and a candidate for the noisy textures.

### MSVC build fixes

Four small fixes so the 4 Nov upstream code builds with MSVC, all matching later upstream commits: the missing `src/audio_core/out` files, and compile errors in `sparse_large_vector.cpp`, `zbic_compression.cpp` and `launch_params.cpp`.

## Building

Same as upstream. See [docs/Build.md](docs/Build.md). Use Visual Studio 2026 Build Tools: the bundled Qt is built with it and won't link with Visual Studio 2022.

To get a GPU crash dump on NVIDIA, configure with `-DENABLE_NSIGHT_AFTERMATH=ON`. This needs the Nsight Aftermath SDK, with the `NSIGHT_AFTERMATH_SDK` environment variable pointing to it. Then enable Nsight Aftermath in the debug settings. Dumps are written to `%APPDATA%\eden\log\gpucrash\`.

## License

GPLv3 or later, same as upstream. See [LICENSE.txt](LICENSE.txt).
