<!--
# SPDX-FileCopyrightText: Copyright 2025 Eden Emulator Project
# SPDX-License-Identifier: GPL-3.0-or-later

# SPDX-FileCopyrightText: 2018 yuzu Emulator Project
# SPDX-License-Identifier: GPL-2.0-or-later
-->

# Eden-DBX-fix

An unofficial fork of the [Eden](https://git.eden-emu.dev/eden-emu/eden/) Nintendo Switch emulator. It contains a small patch aimed at Vulkan "Device loss" crashes seen in Dragon Quest X Offline (0100E2E0152E4000) on an RTX 5060 Laptop GPU.

Everything else is upstream Eden as of 4 Nov 2026. For general documentation, downloads, builds and support, go to the upstream project:
https://git.eden-emu.dev/eden-emu/eden/

Please don't report problems with this fork to the Eden developers.

## Status

The patch has not been tested on hardware yet. It compiles (syntax-checked), but it has not been confirmed to stop the crash.

## Background

This fork started with Dragon Quest X Offline (Japanese, v2.0.1 with the English patch) crashing on Eden v0.2.1. The setup was an i7-14650HX laptop with an RTX 5060 Laptop GPU and 16 GB of RAM. About half the time, the game crashed on a loading screen after 6 to 24 minutes of play. Every crash log ended the same way:

    Render.Vulkan <Critical> vulkan_device.cpp:773:ReportLoss: Device loss occurred!

These were tried first, and none of them stopped the crash:

- Disabling Extended Dynamic State, raising GPU accuracy to High, and lowering Fast GPU Time
- Updating the NVIDIA driver from 591.91 to 617.42 and clearing the shader cache
- Raising the Windows GPU timeout (TdrDelay and TdrDdiDelay set to 10 seconds)
- Removing the English patch, and removing the update that adds the DLC content
- Switching to OpenGL, which didn't boot at all

The same game files run on Ryubing without crashing, only slower. That pointed to a problem in Eden rather than in the game, the patch or the hardware.

The latest Eden master build was worse: it lost the GPU after about 100 seconds. Its log also showed an error that never appeared on v0.2.1:

    core\memory.cpp: assert current_block != 65535

Following that assert through the source led to the `GetSpan` bug described below. It explains why master crashes much sooner. The original v0.2.1 crash may have a separate cause. One open candidate is the dual-GPU laptop issue reported upstream ([Issue-Reports #257](https://github.com/eden-emulator/Issue-Reports/issues/257)), which is why this fork also logs where each device loss is detected.

## What changed

### Memory span contiguity check (`src/core/memory.cpp`)

On current master, the game logs `assert current_block != 65535` in `core/memory.cpp` about 20 seconds in, and later crashes with `Device loss occurred!`.

- Every memory mapping gets a block id from a 16-bit counter. The counter is never reset, so after 65,535 mappings it wraps and unrelated mappings can end up with the same id.
- `GetSpan` treated a range as one contiguous host region if the first page and the page after the end had the same block id. Once ids repeat, that check can pass for ranges that span two separate mappings. The caller then reads or writes host memory that belongs to something else.
- `GetSpan` is used for IPC buffers, audio and NVDEC command buffers, so the bad data can end up on the GPU.

The fix checks every page in the range. A range counts as contiguous only if every page has the same host pointer base and block id. Otherwise `GetSpan` returns `nullptr` and the caller uses the normal copy path. This also fixes an off-by-one: the old check looked at the page after the end of the range instead of the last page.

### Device loss location logging (`src/video_core/...`)

`Device::ReportLoss` now takes the location that detected the loss and adds it to the log line:

- `Device loss occurred! (scheduler submit)`: main GPU command submission
- `Device loss occurred! (present blit submit)`: copying the frame to the swapchain
- `Device loss occurred! (query results)`: reading GPU query results

This makes it easier to tell a rendering fault from a presentation problem, such as the multi-GPU laptop issue reported upstream in [Issue-Reports #257](https://github.com/eden-emulator/Issue-Reports/issues/257).

## Building

Same as upstream. See [docs/Build.md](docs/Build.md).

To get a GPU crash dump on NVIDIA, configure with `-DENABLE_NSIGHT_AFTERMATH=ON`. This needs the Nsight Aftermath SDK, with the `NSIGHT_AFTERMATH_SDK` environment variable pointing to it. Then enable Nsight Aftermath in the debug settings. Dumps are written to `%APPDATA%\eden\log\gpucrash\`.

## License

GPLv3 or later, same as upstream. See [LICENSE.txt](LICENSE.txt).
