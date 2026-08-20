# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `vulkan` ·
llama.cpp `b10488` · `threads=2` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 14.9 | 1.00x | 100% |
| 99 | 12.9 | 0.87x | 87% |

Best: `-ngl 0` at 14.9 tok/s
-- 1.00x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full offload was not best on my machine. CPU-only execution (`-ngl 0`) reached
**14.9 tok/s**, while full Vulkan offload (`-ngl 99`) reached **12.9 tok/s**,
or only 87% of the CPU-only result. This suggests that the Vulkan path incurs
host/device transfer and kernel-launch overhead that outweighs the benefit of
moving this small model to the accelerator. The result does not indicate that
VRAM ran out; it is more consistent with transfer and synchronization overhead
on this low-power Vulkan device.
