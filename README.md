# Vulkan WSI Layer (Sky1 fork)

A Vulkan layer that intercepts window system integration (WSI) calls and
provides swapchain, surface, and presentation support for GPUs whose ICD
lacks native WSI -- such as the Mali vendor driver, which exposes no DRM
render node and therefore cannot use standard Mesa WSI paths.

This is a fork of [Arm's vulkan-wsi-layer](https://gitlab.freedesktop.org/mesa/vulkan-wsi-layer)
via [ginkage's fork](https://github.com/ginkage/vulkan-wsi-layer), which
added X11 MIT-SHM presentation support. Our additions center on
multi-presenter X11 support with automatic routing between three
presentation backends.

## X11 presentation modes

The layer selects an X11 presenter automatically per application:

| Priority | Presenter | Transport | Use case |
|----------|-----------|-----------|----------|
| 1 | Wayland bypass | DMA-BUF via `zwp_linux_dmabuf_v1` | Zink / GL apps under Xwayland |
| 2 | DRI3 | XCB Present extension (COPY) | Native Vulkan apps |
| 3 | SHM | CPU copy via MIT-SHM | Fallback, always available |

**Wayland bypass** detects Xwayland, opens a direct Wayland connection to
the compositor, and presents DMA-BUFs zero-copy. A 2-frame deferred buffer
release ring prevents FBO flicker caused by implicit sync races (compositor
still reading a buffer while the app clears the next frame). Bypass state
is stored in `x11::surface` and persists across swapchain recreations,
avoiding compositor window create/destroy animations.

**DRI3** uses the XCB Present extension with `COPY` semantics. It provides
proper window decorations under Xwayland and is the default for direct
Vulkan applications.

**SHM** performs a CPU-side copy via MIT-SHM shared memory. It serves as
the universal fallback. ARM NEON-optimized copy is enabled automatically on
AArch64.

### Auto-detection

Zink apps are detected via `/proc/self/maps` (presence of `libzink`) or
the `MESA_LOADER_DRIVER_OVERRIDE=zink` environment variable, and are routed
to the Wayland bypass presenter. Direct Vulkan apps receive DRI3. Per-app
overrides can be configured in `/etc/sky1/wsi-routing.conf`.

### Environment variables

| Variable | Effect |
|----------|--------|
| `WSI_NO_WAYLAND_BYPASS=1` | Disable Wayland bypass, fall back to DRI3/SHM |

## Key changes from upstream

- **Multi-presenter routing** -- automatic backend selection with fallback
  chain (bypass -> DRI3 -> SHM)
- **Xwayland bypass** -- zero-copy DMA-BUF presentation to the Wayland
  compositor, bypassing X11 entirely for GL/Zink workloads
- **Deferred buffer release** -- 2-frame ring prevents implicit sync races
  between compositor reads and app rendering
- **Fast swapchain teardown** -- semaphore post on teardown avoids a 250ms
  stall in the page-flip thread
- **Vulkan 1.1 promoted extension injection** -- fixes ICDs that do not
  advertise core-promoted extensions at the device level
- **Non-fatal per-device extension check** -- allows multi-GPU
  configurations where devices expose different extension sets
- **`VK_PRESENT_MODE_IMMEDIATE_KHR`** support for X11 surfaces

## Implemented extensions

Instance extensions:
- `VK_KHR_surface`
- `VK_KHR_xcb_surface` / `VK_KHR_xlib_surface`
- `VK_KHR_wayland_surface`
- `VK_KHR_get_surface_capabilities2`
- `VK_EXT_surface_maintenance1`
- `VK_EXT_headless_surface` (optional)

Device extensions:
- `VK_KHR_swapchain`
- `VK_KHR_shared_presentable_image`
- `VK_EXT_image_compression_control_swapchain`
- `VK_KHR_present_id`
- `VK_EXT_swapchain_maintenance1`

## Building

### Dependencies

- CMake >= 3.4.3
- C++17 compiler
- Vulkan loader and headers (1.1+)
- libdrm
- libwayland-client, wayland-protocols, wayland-scanner
- libxcb, xcb-shm, xcb-sync, xcb-dri3, xcb-present
- libX11, libX11-xcb, libXrandr

### Build

```
mkdir build && cd build
cmake .. \
    -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_WSI_X11=ON \
    -DBUILD_WSI_WAYLAND=ON \
    -DSELECT_EXTERNAL_ALLOCATOR=dma_buf_heaps \
    -DWSIALLOC_MEMORY_HEAP_NAME=system \
    -DENABLE_WAYLAND_FIFO_PRESENTATION_THREAD=ON
make -j$(nproc)
```

### Install

Copy the shared library and JSON manifest into a Vulkan implicit layer
directory:

```
sudo cp libVkLayer_window_system_integration.so \
        VkLayer_window_system_integration.json \
        /usr/share/vulkan/implicit_layer.d/
```

The layer is loaded automatically by the Vulkan loader as an implicit layer.

## Upstream repositories

- **Arm (original):** <https://gitlab.freedesktop.org/mesa/vulkan-wsi-layer>
- **Ginkage (X11 SHM):** <https://github.com/ginkage/vulkan-wsi-layer>

## Related projects

This layer is part of [Sky1 Linux](https://github.com/Sky1-Linux), a Linux
distribution for systems based on the CIX Sky1 / CD8180 SoC.

- [sky1-gpu-support](https://github.com/Sky1-Linux/sky1-gpu-support) -- userspace Mali driver and GPU switcher
- [cix-gpu-kmd](https://github.com/Sky1-Linux/cix-gpu-kmd) -- Mali kernel driver module

## License

MIT. See [LICENSE](LICENSE) for details.
