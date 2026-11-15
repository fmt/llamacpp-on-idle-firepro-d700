# Tested configurations

One row per test. To add yours, open an issue with the "Test report" template or send a pull request that appends a row.

Get the details with:

```bash
sw_vers
# if you use OpenCore Legacy Patcher, note its version too
system_profiler SPHardwareDataType SPDisplaysDataType | grep -E "Model Name|Model Identifier|Memory|Chipset Model|VRAM"
cd ~/llama.cpp && git rev-parse HEAD
```

| Date | Machine | GPUs | RAM | macOS | llama.cpp commit | Backend | Result | Notes |
|---|---|---|---|---|---|---|---|---|
| 2026-11-15 | Mac Pro 6,1 (late 2013) | 2 x FirePro D700, 6 GB each | 64 GB | 15.7.7 (24G720), OCLP 2.5.1 | `e574a17b0` (b12407) | Metal + patch | Works. `GGML_METAL_GPU_INDEX=1` runs on the card without WindowServer ("Slot 1" in Activity Monitor) | Qwen2.5 7B Instruct Q4_K_M, `-ngl 99 -c 4096` |
| 2026-11-15 | Mac Pro 6,1 (late 2013) | 2 x FirePro D700, 6 GB each | 64 GB | 15.7.7 (24G720), OCLP 2.5.1 | `e574a17b0` (b12407) | Metal, no patch | Works, but on the default GPU, the same card that runs WindowServer | Same model and flags |
| 2026-11-15 | Mac Pro 6,1 (late 2013) | 2 x FirePro D700, 6 GB each | 64 GB | 15.7.7 (24G720), OCLP 2.5.1 | `e574a17b0` (b12407) | Vulkan (MoltenVK 1.4.2, vulkan-loader 1.4.363.0, glslang 16.6.0, shaderc 2026.4) | Fails at model load: `compute pipeline creation failed for rms_norm_mul_f32_512` (`rms_norm_f32_512` with fusion disabled) | Both GPUs listed as `Vulkan0` and `Vulkan1` |
