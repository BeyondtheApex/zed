# GPUI on Windows + Overlay Window Support

## 概述

GPUI 是 Zed 编辑器的 GPU 加速 UI 框架。本文档涵盖：
1. GPUI 基本使用
2. GPUI Windows 平台实现分析
3. Overlay 窗口支持的架构与修改

---

## 一、GPUI 基本使用

### 最小示例

```rust
use gpui::*;
use gpui_platform::application;

struct HelloWorld {
    text: SharedString,
}

impl Render for HelloWorld {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .bg(rgb(0x1e1e2e))
            .size_full()
            .justify_center()
            .items_center()
            .text_xl()
            .text_color(rgb(0xcdd6f4))
            .child(format!("Hello, {}!", &self.text))
    }
}

fn main() {
    application().run(|cx: &mut App| {
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::centered(None, size(px(400.), px(300.)), cx),
                )),
                ..Default::default()
            },
            |_, cx| cx.new(|_| HelloWorld { text: "GPUI".into() }),
        )
        .unwrap();
    });
}
```

### 核心概念速查

| 概念 | 说明 |
|---|---|
| `div().child().bg().text_color()` | Flexbox 布局，风格类似 Tailwind CSS |
| `impl Render` | 定义 UI，每次状态变化时被调用 |
| `Entity<T>` | 有状态组件的句柄 |
| `cx.new(\|_\| State { ... })` | 创建 Entity |
| `.on_click(cx.listener(...))` | 鼠标事件 |
| `actions!(...)` + `.on_action(...)` | 键盘/菜单事件 |
| `cx.spawn(async move \|...\| { ... })` | 异步任务 |
| `cx.notify()` | 标记需要重渲染 |
| `.when(cond, \|el\| ...)` | 条件渲染 |
| `.with_animation(id, Animation, \|el, delta\| ...)` | 动画（见下文） |

### 动画

```rust
use std::time::Duration;
use gpui::{Animation, AnimationExt, bounce, ease_in_out, percentage, Transformation};

div()
    .with_animation(
        "my_anim",
        Animation::new(Duration::from_secs(2))
            .repeat()
            .with_easing(bounce(ease_in_out)),
        |element, delta| {
            element.with_transformation(Transformation::rotate(percentage(delta)))
        },
    )
```

缓动函数：`linear`, `ease_in`, `ease_out`, `ease_in_out`, `bounce(ease_in_out)`, 或自定义 `|t: f32| -> f32`。

### 项目依赖配置

```toml
[dependencies]
gpui = { 
    git = "https://github.com/BeyondtheApex/zed", 
    branch = "overlay",
    features = ["overlay"] 
}
gpui_platform = { 
    git = "https://github.com/BeyondtheApex/zed", 
    branch = "overlay",
    features = ["overlay"] 
}
```

> **注意**：`gpui` 和 `gpui_platform` 未独立发布到 crates.io，只能通过 Git 依赖使用。

---

## 二、GPUI Windows 实现分析

### 架构总览

```
┌─────────────────────────────────────────┐
│ gpui (平台无关)                          │
│   Platform trait / Render / Elements     │
│   发布到 crates.io                       │
├─────────────────────────────────────────┤
│ gpui_platform (条件编译分发)              │
│   current_platform() → 各平台实现        │
│   未发布                                 │
├─────────────────────────────────────────┤
│ gpui_windows (Windows 实现)              │
│   WindowsPlatform, WindowsWindow,        │
│   DirectXRenderer, DirectWriteTextSystem │
│   未发布                                 │
└─────────────────────────────────────────┘
```

### Windows 渲染管线

```
GPUI UI (Scene)
  ↓
D3D11 Renderer (DirectXRenderer)
  ├─ Shadows → Quads → Paths (MSAA) → Underlines → Sprites
  ├─ GPU 纹理图集 (DirectXAtlas)
  └─ HLSL 着色器 (shaders.hlsl)
  ↓
CompositionSwapChain
  ├─ FLIP_SEQUENTIAL (flip model)
  ├─ PREMULTIPLIED Alpha
  └─ BufferCount: 3
  ↓
DirectComposition Visual Tree
  ├─ IDCompositionDevice
  ├─ CreateTargetForHwnd
  └─ visual.SetContent(swapchain)
  ↓
DWM 合成 → 屏幕
```

### 关键模块

| 文件 | 职责 |
|---|---|
| `platform.rs` | `WindowsPlatform`：全局状态、事件循环、显示器、跳转列表 |
| `window.rs` | `WindowsWindow`：窗口创建、DPI、全屏、消息处理 |
| `events.rs` | 40+ 种 Win32 消息处理（WM_PAINT, WM_SIZE, WM_KEYDOWN...） |
| `directx_renderer.rs` | D3D11 渲染管线、SwapChain、Present |
| `directx_devices.rs` | D3D11 设备/DXGI 工厂创建、设备丢失恢复 |
| `directx_atlas.rs` | GPU 纹理图集（monochrome/polychrome/subpixel） |
| `direct_write.rs` | DirectWrite 文字排版和渲染 |
| `dispatcher.rs` | Windows 线程池调度 |
| `vsync.rs` | `DwmFlush()` 垂直同步 |
| `shaders.hlsl` | HLSL 着色器 |

### 线程模型

- **主线程**：Win32 消息循环 + GPUI 前台执行器
- **VSync 线程**：等待 DWM 合成节奏后 `RedrawWindow` 触发重绘
- **后台线程池**：Windows `ThreadPool` API，按优先级分发

---

## 三、Overlay 窗口修改

### 设计目标

游戏覆盖层应用需要：
- 透明叠加，不干扰游戏渲染
- 最小化 DWM 合成延迟
- 可选点击穿透
- 可配置的帧呈现策略

### 最终渲染路径

```
GPUI UI
  → D3D11 Renderer
    → CompositionSwapChain (FLIP_SEQUENTIAL, PREMULTIPLIED Alpha)
      → DirectComposition Visual
        → DWM
          → 屏幕
```

### 修改清单（6 个文件，+165 行）

#### 1. `crates/gpui/Cargo.toml` — 新 feature

```toml
overlay = []
```

#### 2. `crates/gpui/src/platform.rs` — 新类型

```rust
#[cfg(all(target_os = "windows", feature = "overlay"))]
pub struct OverlayConfig {
    pub click_through: bool,
    pub present_mode: OverlayPresentMode,
}

#[cfg(all(target_os = "windows", feature = "overlay"))]
pub enum OverlayPresentMode {
    CompositedVSync,                       // Present(1, 0)，DWM 节奏同步
    LowLatency { allow_tearing: bool },    // Present(0, DO_NOT_WAIT)
    EventDriven,                           // 仅 dirty 时 Present(1, 0)
}

pub enum WindowKind {
    Normal,
    PopUp,
    Floating,
    Dialog,
    #[cfg(all(target_os = "windows", feature = "overlay"))]
    Overlay(OverlayConfig),               // ← 新增
}
```

#### 3. `crates/gpui_windows/src/window.rs` — 窗口样式

```rust
// overlay 窗口的 Win32 样式
dwExStyle = WS_EX_TOPMOST
          | WS_EX_NOACTIVATE
          | WS_EX_TOOLWINDOW
          | WS_EX_NOREDIRECTIONBITMAP   // DirectComposition 需要
          | (可选) WS_EX_TRANSPARENT     // 点击穿透

dwStyle   = WS_POPUP                     // 无边框弹出窗口
```

**关键设计决定**：
- **不加 `WS_EX_LAYERED`** — 透明度由 DirectComposition + swapchain `PREMULTIPLIED` alpha 处理。`WS_EX_LAYERED` 是老式 layered window 路径，会干扰 DirectComposition 合成。

#### 4. `crates/gpui_windows/src/directx_renderer.rs` — Present 策略

```rust
#[cfg(feature = "overlay")]
pub(crate) enum PresentMode {
    CompositedVSync,                    // Present(1, 0)
    LowLatency { allow_tearing: bool }, // Present(0, flags)
    EventDriven,                        // 脏检查 + Present(1, 0)
}
```

**ALLOW_TEARING 三重守卫**：
```
系统支持 (CheckFeatureSupport at swapchain creation)
  ∧ 用户开启 (OverlayConfig.present_mode.allow_tearing)
    ∧ SyncInterval == 0 (仅 LowLatency 模式)
      → DXGI_PRESENT_ALLOW_TEARING
否则 → DXGI_PRESENT_DO_NOT_WAIT (SyncInterval=0)
```

**关键设计决定**：
- **不假设 Independent Flip** — CompositionSwapChain 经过 DirectComposition → DWM，不是直接扫描到屏幕。`Flip Model ≠ Independent Flip`。
- **保留 DwmFlush** — VSync 线程不变。`DwmFlush()` 作为默认帧节奏策略（省电、稳定），LowLatency/EventDriven 作为可选替代。
- **CheckFeatureSupport 在 swapchain 创建时调用**，不是在每次 Present 时。

#### 5. `crates/gpui_windows/Cargo.toml` — feature 透传

```toml
overlay = ["gpui/overlay"]
```

#### 6. `crates/gpui_platform/Cargo.toml` — feature 透传

```toml
overlay = ["gpui/overlay", "gpui_windows/overlay"]
```

### 使用示例

```rust
use gpui::*;
use gpui_platform::application;

struct OverlayView;

impl Render for OverlayView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full()
            .bg(rgba(0x00000000))           // 完全透明背景
            .child(
                div()
                    .absolute()
                    .top(px(20.))
                    .right(px(20.))
                    .bg(rgba(0x1e1e2ecc))   // 半透明面板
                    .rounded_lg()
                    .p_4()
                    .text_color(rgb(0xffffff))
                    .child("Game Overlay")
            )
    }
}

fn main() {
    application().run(|cx: &mut App| {
        cx.open_window(
            WindowOptions {
                kind: WindowKind::Overlay(OverlayConfig {
                    click_through: false,
                    present_mode: OverlayPresentMode::CompositedVSync,
                }),
                focus: false,
                show: true,
                ..Default::default()
            },
            |_, cx| cx.new(|_| OverlayView),
        )
        .unwrap();
    });
}
```

### PresentMode 选择指南

| 场景 | 推荐模式 | 原因 |
|---|---|---|
| 静态 HUD（FPS 计数、时钟） | `EventDriven` | 内容不变化时不 Present，零 GPU 开销 |
| 动画覆盖层（图表、滚动） | `CompositedVSync` | DWM 节奏同步，避免撕裂，省电 |
| 实时数据（延迟敏感） | `LowLatency { allow_tearing: false }` | 不阻塞在 VSync，允许超帧率更新 |
| 极低延迟（需实测） | `LowLatency { allow_tearing: true }` | 仅系统支持时启用，可能无实际收益 |

### 待实验验证

| 实验 | 目的 |
|---|---|
| 延迟测量 | 对比三种 PresentMode 在游戏场景下的端到端延迟 |
| ALLOW_TEARING 行为 | 在 composition 路径下是否真的降低延迟 |
| 全屏游戏兼容 | Fullscreen Exclusive vs Borderless Windowed 下的覆盖层可见性 |
| GPU 资源竞争 | 游戏 + 覆盖层同时运行的帧率稳定性 |
| 多显示器/DPI 变化 | WM_DPICHANGED 在 overlay 样式下的正确性 |

---

## 四、维护策略

### 跟随上游

```powershell
# Fork: https://github.com/BeyondtheApex/zed
# Branch: overlay

git fetch upstream
git rebase upstream/main
# 冲突极少：6 个文件，全部 #[cfg(feature = "overlay")] 隔离
git push fork overlay --force-with-lease
```

### Feature gate 保证

所有 overlay 相关代码被 `#[cfg(feature = "overlay")]` 包裹。feature 关闭时：
- 编译路径完全不变
- 无新增代码被编译
- `gpui` 和 `gpui_windows` 行为与上游一致
