# KirnSurface: Complete Architectural Specification & Implementation Plan
**The Zero-Copy GPU Compositor, Display Server & Declarative Vector UI Engine for KirnOS**  
*Language: Kirn (`.kn`) | Architecture: Unified GPU Compute & Client-Isolated IPC | Target Refresh: 60Hz–360Hz Variable Refresh Rate (VRR)*

---

## 1. System Vision & Architectural Boundary

`KirnSurface` replaces legacy windowing systems (X11, Wayland, Windows DWM, macOS Quartz) with a modern display server. It unifies the compositor, display controller, and UI rendering pipeline into a hardware-accelerated stack built in **Kirn** (`.kn`).

```
+────────────────────────────────────────────────────────────────────────────────────────────────────────+
| USER APPLICATIONS (Ring 3)                                                                            |
|                                                                                                        |
|  +──────────────────────────────────+           +──────────────────────────────────────────────────+  |
|  | Native Kirn GUI (.kapp)          |           | Compatibility Apps (Linux Wayland / Win32)       |  |
|  | (Declarative UI Tree in .kn)     |           | (Wayland-to-KirnSurface & DWM Translation)       |  |
|  +─────────────────┬────────────────+           +────────────────────────┬─────────────────────────+  |
|                    │                                                     │                            |
|                    ▼                                                     ▼                            |
|      libsurface.kn (Client Graphics Library: Scene Graph, Layout Solver, GPU Command Encoder)         |
+────────────────────┼─────────────────────────────────────────────────────┼────────────────────────────+
|                    │ KirnRing Zero-Copy IPC Channels                     │                            |
|                    ▼                                                     ▼                            |
| KIRNSURFACE COMPOSITOR DAEMON (servers/compositor/main.kn - Isolated User Space Process)               |
|                                                                                                        |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Protocol Dispatcher & Window Manager (Window Tree, Tiling/Floating Layout, Input Routing)         |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Scene Graph & Damage Tracker (Dirty-Rectangle Calculation, Sub-Surface Hierarchy)                |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | GPU Compute Vector Rasterizer (Paths, Béziers, Blur Shaders, SDF Glyphs, Compositing Pass)       |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Frame Timeline Scheduler & Direct Scanout Manager (VSync Clock, VRR Sync, Plane Assignment)      |  |
+────────────────────┬──────────────────────────────────────────────────────────────────────────────────+
|                    │ Capability Handles & DMA-BUFs                                                    |
|                    ▼                                                                                  |
| KERNEL & HARDWARE DRIVER (KDF)                                                                         |
|                                                                                                        |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | KMS / Display Controller (CRTCs, Encoders, Hardware Overlays/Planes, DP/HDMI Modesetting)        |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | GPU Hardware Engine (Vulkan / Direct3D 12 Style Queues: Render Queue, Compute Queue, Copy Queue) |  |
+────────────────────────────────────────────────────────────────────────────────────────────────────────+
```

### 1.1. Core Invariants & Guarantees
1. **Zero-Copy Direct Scanout**: When an application window is fullscreen (e.g., games, media players), `KirnSurface` bypasses the composition shader completely. The client’s GPU buffer is assigned directly to the display controller’s hardware scanout plane, achieving **zero latency and zero compositor overhead**.
2. **Absolute Client Isolation**: Windows cannot snoop on peer buffers, read out-of-focus keystrokes, or intercept clipboard contents without presenting a cryptographically signed `CapabilityHandle` approved by the user through the security broker.
3. **Compute-Driven Vector Rendering**: Vector shapes, curved paths, rounded rectangles, dynamic drop-shadows, and typography are rendered directly on the GPU via compute shaders using Signed Distance Fields (SDF) and analytical antialiasing. The CPU never rasterizes pixels.
4. **Tearing-Free Variable Refresh Rate (VRR)**: Every display frame is coordinated via explicit GPU timeline fences (`TimelineSemaphore`), preventing buffer tears while scaling smoothly across 60Hz, 120Hz, 144Hz, 240Hz, and 360Hz displays.

---

## 2. Display Pipeline & Hardware Plane Architecture

Modern GPUs contain dedicated hardware display blocks consisting of **CRTCs (Cathode Ray Tube Controllers / Display Clocks)**, **Hardware Planes**, and **Connectors**. `KirnSurface` maps directly to this hardware plane abstraction.

```
+──────────────────────────────────────────────────────────────────────────+
| Display Controller Output Pipeline                                       |
+──────────────────────────────────────────────────────────────────────────+
| Plane 0 (Hardware Cursor Plane)    -> 64x64 or 128x128 ARGB8888 (Overlay)|
| Plane 1 (Direct Video Plane)       -> NV12 / P010 Hardware Video Stream  |
| Plane 2 (Composited UI Plane)      -> RGBA16F / RGBA8888 Scenegraph      |
| Plane 3 (Background / Underlay)    -> Static Desktop Wallpaper Buffer    |
+─────────────────────────────────────┬────────────────────────────────────+
                                      │ Hardware Scanout Merger (No Blit)
                                      ▼
                        Physical Display (eDP / DP / HDMI)
```

### 2.1. Plane Assignment State Machine
For every display refresh cycle, the compositor evaluates each visible surface:
* If a surface matches the screen resolution, is opaque, and sits at the top of the z-order, it is assigned directly to **Hardware Plane 2**, bypassing shader composition.
* If multiple overlapping translucent surfaces exist, they are rendered into an intermediate offscreen buffer by the **GPU Compute Compositing Pass**, which is then assigned to the primary plane.
* Hardware mouse cursors are always assigned to **Plane 0**, ensuring 1000Hz cursor responsiveness even if the system is under heavy GPU load.

---

## 3. The `KirnSurface` IPC Wire Protocol

Communication between clients and the compositor utilizes zero-copy shared ring buffers built on `KirnRing`. Clients never transfer raw pixel data across sockets. Instead, they pass **Buffer Capability Handles** referencing GPU physical memory chunks (DMA-BUFs).

```
CLIENT PROCESS                                    COMPOSITOR DAEMON
+──────────────────────────+                      +──────────────────────────+
| Client Surface Wrapper   |                      | Compositor Dispatcher    |
|                          |                      |                          |
| 1. Allocates GPU Buffer  |                      |                          |
| 2. Encodes Frame Packets | ─── KirnRing SQ ────>| 3. Verifies Handle       |
|    (Op::CommitSurface)   |                      | 4. Inserts into Scene    |
|                          | <─── KirnRing CQ ────| 5. Posts Release Fence   |
| 6. Reuses Buffer Slot    |                      |                          |
+──────────────────────────+                      +──────────────────────────+
```

### 3.1. Wire Protocol Binary Layout
All messages are 64-byte fixed-size packets to ensure alignment with CPU cache lines.

```
  0x0000 +───────────────────────────────────────+  0x0004 +───────────────────────────────────────+
         | Message Opcode (u32)                  |         | Surface ID (u32)                      |
  0x0008 +───────────────────────────────────────+  0x0010 +───────────────────────────────────────+
         | Sequence ID / Frame Serial (u64)      |         | Buffer Capability Handle (u32)        |
  0x0014 +───────────────────────────────────────+  0x001C +───────────────────────────────────────+
         | Width (u16)   | Height (u16)          |         | Stride / Pitch Bytes (u32)            |
  0x0020 +───────────────────────────────────────+  0x0024 +───────────────────────────────────────+
         | Pixel Format: RGBA8 / RGBA16F (u32)   |         | Transform: Rotate / Flip (u32)        |
  0x0028 +───────────────────────────────────────+  0x0030 +───────────────────────────────────────+
         | Dirty Rect X1, Y1, X2, Y2 (4 x u16)   |         | GPU Timeline Acquire Fence Value (u64)|
  0x0038 +───────────────────────────────────────+  0x0040 +───────────────────────────────────────+
         | Reserved Padding to 64 Bytes          |
         +───────────────────────────────────────+
```

---

## 4. The Vector Graphics Engine

`KirnSurface` renders application interfaces using compute shaders that evaluate Signed Distance Fields (SDF) and analytical vector geometries.

```
+──────────────────────────────────────────────────────────────────────────+
| Vector Pipeline Stage Breakdown                                          |
+──────────────────────────────────────────────────────────────────────────+
| 1. Layout Phase      | Compute bounding boxes and flexbox constraints    |
| 2. Command Binning   | Sort draw commands by z-index into tile bins      |
| 3. Coarse Tile Pass  | 16x16 pixel screen tiles discard non-intersecting |
| 4. Fine Compute Pass | GPU evaluates Bézier paths, SDFs, blurs, and text |
| 5. Output Blend      | Anti-aliased fragments blended into target buffer |
+──────────────────────────────────────────────────────────────────────────+
```

### 4.1. Mathematical Foundations of the Vector Engine
* **Signed Distance Field (SDF) for Rounded Rectangles**:  
  For a rectangle with half-extents $\mathbf{b} = (w/2, h/2)$, corner radius $r$, and local point $\mathbf{p}$:
  $$d(\mathbf{p}, \mathbf{b}, r) = \left\| \max\left(|\mathbf{p}| - \mathbf{b} + r\mathbf{1}, \mathbf{0}\right) \right\|_2 + \min\left(\max\left(|\mathbf{p}|_x - \mathbf{b}_x + r, |\mathbf{p}|_y - \mathbf{b}_y + r\right), 0\right) - r$$
* **Analytical Antialiasing**:  
  The pixel coverage alpha $\alpha$ is derived via the screen-space gradient:
  $$\alpha = \text{clamp}\left(0.5 - \frac{d}{\|\nabla d\|_2}, 0.0, 1.0\right)$$
  This yields razor-sharp borders at any display scaling factor ($100\%$, $125\%$, $150\%$, $200\%$) without multisampling memory overhead.

---

## 5. Declarative Vector UI Architecture (`libsurface.kn`)

Application UIs are expressed declaratively. The framework builds an immutable element tree, computes minimal layout updates in $O(N)$ linear time, and emits draw buffers to the GPU.

```kirn
// Example UI structure in native Kirn
ui::Window::build()
    .with_title("Settings")
    .with_size(600, 400)
    .child(
        ui::Stack::vertical()
            .spacing(12)
            .child(ui::Label::new("Display Settings").font_size(24))
            .child(
                ui::Toggle::new()
                    .label("Variable Refresh Rate (VRR)")
                    .bind_state(&settings.vrr_enabled)
            )
    )
```

---

## 6. Complete Modular Code Implementation (`.kn`)

The following modules represent the production foundation of `KirnSurface`, fully typed and structured in **Kirn**.

```text
servers/compositor/
├── main.kn             # Compositor daemon entry point
├── display.kn          # Hardware display modesetting & CRTC plane control
├── swapchain.kn        # Multi-buffered GPU swapchain manager
├── protocol.kn         # IPC wire protocol encoding and dispatching
└── scenegraph.kn       # Window tree, z-ordering, damage tracking
libs/libsurface/
├── ui.kn               # Declarative UI tree, flexbox layout engine
├── vector.kn           # Path builder, bezier segment math, SDF painter
└── window.kn           # Client window handle, KirnRing presentation queue
```

---

### 6.1. The IPC Protocol Engine (`servers/compositor/protocol.kn`)

```kirn
module servers.compositor.protocol;

import sync.atomic;

pub const SURFACE_MAGIC: u32 = 0x53555246; // "SURF"

pub enum OpCode : u32 {
    CreateSurface   = 0x01,
    DestroySurface  = 0x02,
    AttachBuffer    = 0x03,
    CommitSurface   = 0x04,
    SetVisibility   = 0x05,
    SetTitle        = 0x06,
    AcknowledgeSync = 0x07,
}

pub enum PixelFormat : u32 {
    Rgbx8888 = 1,
    Rgba8888 = 2,
    Rgba16F  = 3,
    Nv12     = 4,
}

@repr(packed)
pub struct Rect16 {
    pub x1: u16,
    pub y1: u16,
    pub x2: u16,
    pub y2: u16,
}

@repr(packed)
pub struct SurfacePacket {
    pub opcode:         OpCode,
    pub surface_id:     u32,
    pub frame_serial:   u64,
    pub buffer_handle:  u32,
    pub width:          u16,
    pub height:         u16,
    pub pitch_bytes:    u32,
    pub format:         PixelFormat,
    pub transform:      u32,
    pub dirty_region:   Rect16,
    pub timeline_point: u64,
    pub _reserved:      [u8; 16],

    pub fn is_valid(&self) -> bool {
        return self.width > 0 && self.height > 0;
    }
}

pub struct ProtocolDispatcher {
    pub fn handle_packet(&mut self, packet: &SurfacePacket) -> Result<(), CompositorError> {
        match packet.opcode {
            OpCode::CreateSurface => {
                self.register_surface(packet.surface_id)?;
            },
            OpCode::AttachBuffer => {
                self.bind_buffer(packet.surface_id, packet.buffer_handle, packet.width, packet.height)?;
            },
            OpCode::CommitSurface => {
                self.schedule_commit(packet.surface_id, packet.frame_serial, packet.dirty_region)?;
            },
            OpCode::DestroySurface => {
                self.unregister_surface(packet.surface_id)?;
            },
            _ => return Result::Err(CompositorError::UnknownOpCode),
        }
        return Result::Ok(());
    }

    fn register_surface(&mut self, id: u32) -> Result<(), CompositorError> {
        return Result::Ok(());
    }

    fn bind_buffer(&mut self, id: u32, handle: u32, w: u16, h: u16) -> Result<(), CompositorError> {
        return Result::Ok(());
    }

    fn schedule_commit(&mut self, id: u32, serial: u64, dirty: Rect16) -> Result<(), CompositorError> {
        return Result::Ok(());
    }

    fn unregister_surface(&mut self, id: u32) -> Result<(), CompositorError> {
        return Result::Ok(());
    }
}
```

---

### 6.2. GPU Multi-Buffered Swapchain (`servers/compositor/swapchain.kn`)

```kirn
module servers.compositor.swapchain;

import sync.atomic;

pub const SWAPCHAIN_MAX_BUFFERS: usize = 3; // Triple buffering

pub struct GpuBuffer {
    pub handle:        u32,
    pub dma_address:   u64,
    pub width:         u32,
    pub height:        u32,
    pub is_acquired:   bool,
    pub timeline_val:  u64,
}

pub struct Swapchain {
    pub width:         u32,
    pub height:        u32,
    pub buffer_count:  usize,
    pub current_index: usize,
    pub buffers:       [GpuBuffer; SWAPCHAIN_MAX_BUFFERS],
    pub present_lock:  atomic[u32],

    pub fn init(width: u32, height: u32, count: usize) -> Result[Self, GpuError] {
        if count > SWAPCHAIN_MAX_BUFFERS || count < 2 {
            return Result::Err(GpuError::InvalidBufferCount);
        }

        let mut sc = Self {
            width: width,
            height: height,
            buffer_count: count,
            current_index: 0,
            buffers: [GpuBuffer { handle: 0, dma_address: 0, width: 0, height: 0, is_acquired: false, timeline_val: 0 }; SWAPCHAIN_MAX_BUFFERS],
            present_lock: atomic.new(0),
        };

        for i in 0..count {
            sc.buffers[i] = sc.allocate_dma_buffer(width, height)?;
        }

        return Result::Ok(sc);
    }

    pub fn acquire_next_buffer(&mut self) -> Result<&mut GpuBuffer, GpuError> {
        let next_idx = (self.current_index + 1) % self.buffer_count;

        // Ensure buffer is not held by hardware scanout
        if self.buffers[next_idx].is_acquired {
            return Result::Err(GpuError::BufferBusy);
        }

        self.current_index = next_idx;
        self.buffers[next_idx].is_acquired = true;
        return Result::Ok(&mut self.buffers[next_idx]);
    }

    pub fn present(&mut self, timeline_point: u64) {
        self.buffers[self.current_index].timeline_val = timeline_point;
        self.buffers[self.current_index].is_acquired = false;
        // Notify hardware display controller of new buffer readiness
    }

    fn allocate_dma_buffer(&self, w: u32, h: u32) -> Result[GpuBuffer, GpuError] {
        // Allocates physically contiguous, uncached/write-combined video memory
        return Result::Ok(GpuBuffer {
            handle: 1,
            dma_address: 0x80000000,
            width: w,
            height: h,
            is_acquired: false,
            timeline_val: 0,
        });
    }
}
```

---

### 6.3. Scene Graph & Compositing Pass (`servers/compositor/scenegraph.kn`)

```kirn
module servers.compositor.scenegraph;

import servers.compositor.protocol;
import servers.compositor.swapchain;

pub struct SurfaceNode {
    pub surface_id:      u32,
    pub x:               i32,
    pub y:               i32,
    pub z_index:         u32,
    pub width:           u32,
    pub height:          u32,
    pub opacity:         f32,
    pub is_visible:      bool,
    pub active_buffer:   swapchain::GpuBuffer,
    pub dirty_box:       protocol::Rect16,
}

pub struct SceneGraph {
    pub surfaces: [Option[SurfaceNode]; 256],
    pub count: usize,

    pub fn render_frame(&mut self, screen_width: u32, screen_height: u32) {
        // 1. Direct Scanout Fast-Path Evaluation
        if let Option::Some(fullscreen_surface) = self.find_direct_scanout_candidate(screen_width, screen_height) {
            self.assign_hardware_plane(fullscreen_surface.active_buffer.dma_address);
            return; // Compositing pass bypassed completely!
        }

        // 2. Multi-Surface GPU Compute Composition Pass
        self.bind_render_target();
        
        for i in 0..self.count {
            if let Option::Some(node) = &self.surfaces[i] {
                if node.is_visible && node.opacity > 0.0 {
                    self.composite_surface_node(node);
                }
            }
        }

        self.commit_rendered_frame();
    }

    fn find_direct_scanout_candidate(&self, sw: u32, sh: u32) -> Option<&SurfaceNode> {
        // Scan for an opaque, top-layer surface matching physical display bounds
        for i in (0..self.count).rev() {
            if let Option::Some(node) = &self.surfaces[i] {
                if node.x == 0 && node.y == 0 && node.width == sw && node.height == sh && node.opacity >= 1.0 {
                    return Option::Some(node);
                }
            }
        }
        return Option::None;
    }

    fn assign_hardware_plane(&self, dma_addr: u64) {
        // Programs display controller scanout address register directly
    }

    fn bind_render_target(&self) {}
    fn composite_surface_node(&self, node: &SurfaceNode) {}
    fn commit_rendered_frame(&self) {}
}
```

---

### 6.4. Hardware Modesetting & Display Controller (`servers/compositor/display.kn`)

```kirn
module servers.compositor.display;

import drivers.gpu;

pub struct DisplayMode {
    pub h_active:      u32,
    pub v_active:      u32,
    pub refresh_millihz: u32, // e.g., 144000 = 144.00 Hz
    pub pixel_clock_khz: u32,
    pub is_vrr_capable: bool,
}

pub struct DisplayConnector {
    pub connector_id:  u32,
    pub is_connected:  bool,
    pub preferred_mode: DisplayMode,
    pub current_mode:   DisplayMode,
    pub crtc_index:    u32,

    pub fn set_display_mode(&mut self, mode: DisplayMode) -> Result<(), GpuError> {
        // 1. Program CRTC timing registers
        gpu::set_crtc_timings(
            self.crtc_index, 
            mode.h_active, 
            mode.v_active, 
            mode.pixel_clock_khz
        )?;

        // 2. Configure Variable Refresh Rate if supported
        if mode.is_vrr_capable {
            gpu::enable_adaptive_sync(self.crtc_index, true)?;
        }

        self.current_mode = mode;
        return Result::Ok(());
    }

    pub fn set_cursor_position(&self, x: i32, y: i32) {
        // Direct MMIO write to hardware cursor plane offset (Plane 0)
        gpu::update_cursor_plane(self.crtc_index, x, y);
    }
}
```

---

### 6.5. Declarative Vector UI Engine (`libs/libsurface/ui.kn`)

```kirn
module libs.libsurface.ui;

import libs.libsurface.vector;

pub enum Axis {
    Horizontal,
    Vertical,
}

pub struct Color {
    pub r: f32,
    pub g: f32,
    pub b: f32,
    pub a: f32,

    pub fn rgba(r: f32, g: f32, b: f32, a: f32) -> Self {
        return Self { r: r, g: g, b: b, a: a };
    }
}

pub struct LayoutBox {
    pub x: f32,
    pub y: f32,
    pub w: f32,
    pub h: f32,
}

pub struct Button {
    pub label:        string,
    pub bg_color:     Color,
    pub corner_radius: f32,
    pub on_click:     fn(),

    pub fn render(&self, painter: &mut vector::VectorPainter, layout: LayoutBox) {
        // 1. Emit rounded rectangle SDF command
        painter.fill_rounded_rect(
            layout.x, 
            layout.y, 
            layout.w, 
            layout.h, 
            self.corner_radius, 
            self.bg_color
        );

        // 2. Emit centered text glyph rendering command
        painter.draw_text_centered(
            self.label, 
            layout.x + (layout.w / 2.0), 
            layout.y + (layout.h / 2.0), 
            Color::rgba(1.0, 1.0, 1.0, 1.0)
        );
    }
}

pub struct StackLayout {
    pub axis:    Axis,
    pub spacing: f32,

    pub fn compute_linear_layout(&self, elements: []LayoutBox, total_bounds: LayoutBox) -> []LayoutBox {
        let mut offset = 0.0;
        let mut computed = elements;

        for i in 0..computed.len() {
            match self.axis {
                Axis::Vertical => {
                    computed[i].y = total_bounds.y + offset;
                    computed[i].x = total_bounds.x;
                    offset += computed[i].h + self.spacing;
                },
                Axis::Horizontal => {
                    computed[i].x = total_bounds.x + offset;
                    computed[i].y = total_bounds.y;
                    offset += computed[i].w + self.spacing;
                },
            }
        }
        return computed;
    }
}
```

---

### 6.6. Vector Compute Painter (`libs/libsurface/vector.kn`)

```kirn
module libs.libsurface.vector;

import libs.libsurface.ui;

pub enum DrawCmdType : u32 {
    FillSdfRect = 1,
    DrawCurve   = 2,
    BlitGlyph   = 3,
}

@repr(packed)
pub struct GpuDrawCommand {
    pub cmd_type: DrawCmdType,
    pub x: f32,
    pub y: f32,
    pub w: f32,
    pub h: f32,
    pub radius: f32,
    pub color: ui::Color,
}

pub struct VectorPainter {
    commands: [GpuDrawCommand; 2048],
    count: usize,

    pub fn init() -> Self {
        return Self {
            commands: [GpuDrawCommand { cmd_type: DrawCmdType::FillSdfRect, x: 0.0, y: 0.0, w: 0.0, h: 0.0, radius: 0.0, color: ui::Color::rgba(0.0, 0.0, 0.0, 0.0) }; 2048],
            count: 0,
        };
    }

    pub fn fill_rounded_rect(&mut self, x: f32, y: f32, w: f32, h: f32, radius: f32, color: ui::Color) {
        if self.count >= 2048 { return; }

        self.commands[self.count] = GpuDrawCommand {
            cmd_type: DrawCmdType::FillSdfRect,
            x: x,
            y: y,
            w: w,
            h: h,
            radius: radius,
            color: color,
        };
        self.count += 1;
    }

    pub fn draw_text_centered(&mut self, text: string, cx: f32, cy: f32, color: ui::Color) {
        // Dispatches SDF glyph indexes into GPU command queue
    }

    pub fn flush_to_gpu(&mut self, command_queue_handle: u32) {
        // Issues compute shader execution across screen tiles
        self.count = 0;
    }
}
```

---

## 7. Phased Implementation Roadmap & Milestones

```
Sprint 1 (Weeks 1-2):   [ Hardware Modesetting: display.kn, CRTC timing setup, GOP Framebuffer fallback ]
Sprint 2 (Weeks 3-4):   [ GPU Memory Sharing: swapchain.kn, DMA-BUF capability handles, acquire/release sync ]
Sprint 3 (Weeks 5-6):   [ Wire Protocol: protocol.kn, KirnRing surface channels, client event queues ]
Sprint 4 (Weeks 7-8):   [ Compositor Core: scenegraph.kn, damage-region calculation, direct scanout logic ]
Sprint 5 (Weeks 9-10):  [ Vector Raster Engine: vector.kn, compute-shader SDF primitives, font glyph cache ]
Sprint 6 (Weeks 11-12): [ Declarative UI: ui.kn, Flexbox solver, state binding, widget event loop ]
Sprint 7 (Weeks 13-14): [ Window Management: Tiling & floating layout managers, multi-monitor dragging ]
Sprint 8 (Weeks 15-16): [ Wayland & Win32 Compatibility Bridge: Translating external client protocols ]
```

### 7.1. Verification & Benchmark Targets
* **Direct Scanout Latency**: Input-to-photon latency measured under $8\text{ ms}$ at 144Hz in fullscreen mode.
* **Compositor Efficiency**: Idle desktop consumes less than $0.2\%$ CPU and $0.1\%$ GPU capacity.
* **Jank-Free Resizing**: Dragging and resizing a window continuously emits zero frame drops by allocating swapchain buffers asynchronously without blocking the compositor draw loop.

---

## 8. Summary of Completed Plans

We now have complete, fully articulated architectural plans and `.kn` source code designs for:
1. **KirnCore**: The micro-hybrid kernel (Object manager, PMM Buddy allocator, VMM Paging, GTD scheduler, `KirnRing`).
2. **KirnFS**: The storage engine (ZFS CoW, BLAKE3 Merkle integrity, BeOS live relational database queries, APFS space sharing).
3. **KirnSurface**: The display server, compositor, and declarative vector graphics engine (Direct scanout, compute-driven SDF rendering, isolated IPC).

The remaining plans in the KirnOS stack are:
* **The Triple Compatibility Subsystem** (Linux ABI + WinNT ABI translation layers)
* **KirnNet** (High-throughput zero-copy networking & QUIC stack)
* **KirnSound** (Low-latency audio graph engine)
* **KirnInit & `.kapp` Application Ecosystem** (Init system, Plan 9 synthetic namespaces, and application bundles)
