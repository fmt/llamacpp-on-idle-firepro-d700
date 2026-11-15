# llama.cpp on one FirePro D700 of a Mac Pro 6,1

How to run a local LLM with [llama.cpp](https://github.com/ggml-org/llama.cpp) on a single AMD FirePro D700 of an Intel Mac Pro 6,1 (late 2013), keeping the other card free for the display. Includes a small patch that lets the Metal backend choose which GPU to use, and a record of why the Vulkan route (MoltenVK) failed.

## Status

This was tested once, on one machine, at one point in time. It may not apply or work later, because llama.cpp changes quickly and the patch edits one specific line of the Metal backend. No support is promised. Reports from other machines are welcome (see [Contributing](#contributing)).

## Tested configuration

| | |
|---|---|
| Machine | Mac Pro 6,1 (late 2013), 64 GB RAM |
| GPUs | 2 x AMD FirePro D700, 6 GB VRAM each |
| macOS | 15.7.7 (build 24G720), installed with OpenCore Legacy Patcher 2.5.1 (the Mac Pro 6,1 is not officially supported on this version) |
| llama.cpp | build b12407, commit `e574a17b062be51b98e239ccd3d3743e3d902333` |
| Backend that works | Metal, with the patch in this repo |
| Backend that fails | Vulkan through MoltenVK |
| Model | Qwen2.5 7B Instruct, Q4_K_M (about 4.7 GB) |
| Homebrew status | Intel Mac, unsupported (no bottles, formulas build from source) |

Full details are in [TESTED_ON.md](TESTED_ON.md).

## Summary

- Metal works. Vulkan builds and lists both GPUs but crashes when the model loads.
- Upstream, the Metal backend always uses the macOS default GPU, which on this machine is the card that also runs WindowServer. There is no flag to pick the other card.
- The patch adds an environment variable, `GGML_METAL_GPU_INDEX`, that selects a GPU by index. With it, the model ran on the idle card (shown as "Slot 1" in Activity Monitor) while WindowServer stayed on the other.

## Quick start

### 1. Prerequisites

```bash
xcode-select --install   # skip if already installed
brew install cmake
```

No Vulkan, MoltenVK or shader packages are needed.

### 2. Clone and check out the tested commit

```bash
git clone https://github.com/ggml-org/llama.cpp ~/llama.cpp
cd ~/llama.cpp
git checkout e574a17b062be51b98e239ccd3d3743e3d902333
```

Using a newer commit may work (see the note on newer versions below), but this is the one the guide was verified on.

### 3. Apply the patch

From the `llama.cpp` directory, with this repository cloned next to it or the patch file copied over:

```bash
git apply --check /path/to/this-repo/patches/0001-metal-select-gpu-by-index.patch
git apply /path/to/this-repo/patches/0001-metal-select-gpu-by-index.patch
```

If `--check` fails, llama.cpp has moved on. Look at the patch (it replaces one line, `dev->mtl_device = MTLCreateSystemDefaultDevice();` in `ggml/src/ggml-metal/ggml-metal-device.m`) and make the same change by hand.

### 4. Build with Metal

```bash
cmake -B build-metal -DGGML_METAL=ON -DGGML_VULKAN=OFF
cmake --build build-metal --config Release -j
./build-metal/bin/llama-cli --list-devices
```

The device list shows one entry, for example `MTL0: AMD Radeon HD - FirePro D700 (6144 MiB, ...)`. That is the default GPU only. The patch prints every GPU it finds at model load, as lines like `GPU index 0: AMD Radeon HD - FirePro D700`.

### 5. Download the model

```bash
mkdir -p ~/ai_models && cd ~/ai_models
curl -L -O "https://huggingface.co/bartowski/Qwen2.5-7B-Instruct-GGUF/resolve/main/Qwen2.5-7B-Instruct-Q4_K_M.gguf"
ls -lh Qwen2.5-7B-Instruct-Q4_K_M.gguf   # about 4.7 GB
```

### 6. Run on a chosen GPU

Both cards have the same name, so find the right index by trying each and watching Activity Monitor (GPU tab, or Window > GPU History). The index that puts `llama-cli` under the card that is not running WindowServer is the one to keep. On the tested machine this was index 1, shown as "Slot 1".

```bash
cd ~/llama.cpp
GGML_METAL_GPU_INDEX=1 ./build-metal/bin/llama-cli \
  -m ~/ai_models/Qwen2.5-7B-Instruct-Q4_K_M.gguf \
  -ngl 99 -c 4096
```

To make it permanent:

```bash
echo 'export GGML_METAL_GPU_INDEX=1' >> ~/.zshrc
source ~/.zshrc
echo $GGML_METAL_GPU_INDEX
```

If the variable is unset or out of range, the patch falls back to the default GPU, so behaviour matches upstream.

Browser UI instead of the terminal chat:

```bash
./build-metal/bin/llama-server \
  -m ~/ai_models/Qwen2.5-7B-Instruct-Q4_K_M.gguf \
  -ngl 99 -c 4096 --port 8080
```

Then open `http://localhost:8080`.

## Flags

| Flag | Meaning |
|---|---|
| `-ngl 99` | Offload all layers to the GPU |
| `-c 4096` | Context size; lower to 2048 if memory is tight |
| `-fa off` | Turn off flash attention if the output is garbled |

Recent builds start in chat mode by default and reject `-cnv`.

## Newer llama.cpp versions

The patch passes `git apply --check` against master as of 2026-11-15 (commit `de32b268c4ad55d45f3b7b16c029f4e077f4ba74`), but it was only built and run on the pinned commit above. To update an existing checkout:

```bash
cd ~/llama.cpp
git checkout ggml/src/ggml-metal/ggml-metal-device.m
git pull
git apply /path/to/this-repo/patches/0001-metal-select-gpu-by-index.patch
cmake --build build-metal --config Release -j
```

## Observed versus suspected

### Observed (seen on the tested machine)

- With Vulkan (MoltenVK 1.4.2 and vulkan-loader 1.4.363.0 from Homebrew), llama.cpp compiles and `--list-devices` shows `Vulkan0` and `Vulkan1`, 6144 MiB each.
- Loading the model then aborts with:

  ```
  ggml_vulkan: compute pipeline creation failed for rms_norm_mul_f32_512:
  vk::Device::createComputePipeline: ErrorInitializationFailed
  ```

- With `GGML_VK_DISABLE_FUSION=1 GGML_VK_DISABLE_F16=1` set, the failure moved from the fused kernel to the plain one, which every model uses, so there was nothing left to switch off:

  ```
  ggml_vulkan: compute pipeline creation failed for rms_norm_f32_512:
  vk::Device::createComputePipeline: ErrorInitializationFailed
  ```

  The move to the plain kernel is consistent with fusion being disabled. Whether the second variable name is honoured by that build was not verified.
- The Metal backend builds, lists one device (`MTL0`) and runs the model. Without the patch, Activity Monitor shows the process on the same card as WindowServer.
- Reading the source at the tested commit: `GGML_METAL_DEVICES` only sets how many devices are created, and device initialisation always calls `MTLCreateSystemDefaultDevice()`.
- With the patch and `GGML_METAL_GPU_INDEX=1`, Activity Monitor shows `llama-cli` under the other card ("Slot 1").

### Suspected (not verified)

- **Cause of the Vulkan failure.** On macOS, Vulkan is not native. llama.cpp's Vulkan backend ships its shaders as SPIR-V, and MoltenVK translates them to Metal shaders and hands them to the Metal compiler. The error appears at pipeline creation, which is where that last step happens. The likely cause is that the Metal compiler for the old AMD (GCN) driver rejects the translated shaders. The underlying compiler message was not captured, so the exact construct is unknown. If you capture it, please open an issue. MoltenVK has logging controls (`MVK_CONFIG_LOG_LEVEL`, name from memory) that may help.
- **Why Metal works.** It uses kernels written directly for Metal, compiled by the same Metal compiler, with no translation layer. This is a reasoned explanation, not something that was tested.
- **Effect of OpenCore Legacy Patcher.** The tested macOS version is not officially supported on this hardware, so the GPU driver stack is restored by OCLP root patches. Whether the MoltenVK failure comes from that patched stack, or would also happen on a macOS version Apple supports on this Mac, was not tested.
- **Other GPUs.** Whether D300 and D500 cards, or other AMD GPUs on Intel Macs, behave the same way is unknown.
- **Index order.** Whether the GPU index order is the same across machines, reboots and display setups is unknown. Always check with Activity Monitor.

### Trade-off

| | Vulkan (MoltenVK) | Metal |
|---|---|---|
| Runs on the D700 | No (pipeline creation fails) | Yes |
| Pick a specific GPU | Yes (`--device Vulkan1`) | Not upstream; needs the patch in this repo |
| Extra packages | MoltenVK, loader, shader tools, headers | None beyond cmake |

## Problems met while setting up Vulkan

| Problem | Cause and fix |
|---|---|
| `vulkan/vulkan_core.h` not found | Headers were not installed; cloning `KhronosGroup/Vulkan-Headers` and setting `CPATH` fixed it |
| `vulkan-loader` and `molten-vk` missing | Installed with `brew install molten-vk vulkan-loader` |
| `vulkaninfo: command not found` | `vulkan-tools` needs a full Xcode install to build on Intel; skipped |
| `-cnv` rejected | Flag removed in recent builds; chat mode is the default |
| Homebrew warns about Intel support | Bottles are no longer built for Intel Macs, so formulas compile from source |

## Troubleshooting

| Symptom | Fix |
|---|---|
| Process still shows under the display card | Try the other index (0 or 1) and recheck Activity Monitor |
| Garbled text | Add `-fa off` and try `-c 2048` |
| Out of memory | Use a smaller quant (Q3_K_M) or a 3B model |
| Patch does not apply | llama.cpp has changed; make the one-line change by hand (see step 3) |
| Patched build fails to compile | Revert with `git checkout ggml/src/ggml-metal/ggml-metal-device.m` and use the default GPU |
| Metal fails to start | Build CPU-only to isolate the cause: `cmake -B build-cpu -DGGML_METAL=OFF -DGGML_VULKAN=OFF` |

## Contributing

Results from other machines are the most useful contribution. Open an issue with the "Test report" template, or add a row to [TESTED_ON.md](TESTED_ON.md) in a pull request. Include the output of `sw_vers`, whether you use OpenCore Legacy Patcher (and which version), the llama.cpp commit, the GPU model and what happened.

The patch is a local workaround. A proper upstream change (a documented device index, or listing every Metal GPU in `--list-devices`) would make this repository unnecessary.

## License

The text and patch in this repository are under the MIT License (see [LICENSE](LICENSE)). The patch applies to llama.cpp, which is also MIT licensed.
