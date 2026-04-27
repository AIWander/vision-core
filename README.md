# vision-core

Shared OCR and image analysis library for CPC MCP servers.

## What it does

Provides Windows OCR, image capture, template matching, and vision primitives used by the `hands` server as its vision-fallback layer. Centralizes image analysis so multiple CPC servers can share a single, tested implementation.

## Key features

- **Windows OCR** — wraps `Windows.Media.Ocr` for high-accuracy on-screen text extraction
- **Screenshot capture** — full-screen and region capture via the `screenshots` crate
- **Image diff** — pixel-level comparison for visual change detection
- **Template matching** — find UI elements by image template within a larger screenshot
- **Base64 encode/decode** — helpers for passing image data over JSON-RPC
- **Async-first** — built on Tokio; all blocking WinRT calls are offloaded appropriately

## Usage

Add as a path dependency from another CPC server crate:

```toml
[dependencies]
vision-core = { git = "https://github.com/AIWander/vision-core", tag = "v0.1.0" }
```

```rust
use vision_core::ocr;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let text = ocr::recognize_screen().await?;
    println!("{}", text);
    Ok(())
}
```

## Architecture

`vision-core` is a library crate (no binary). It is consumed by:

- **hands** (`browser-mcp`) — vision-fallback layer for UI automation when a11y trees are unavailable
- Future CPC servers that need on-screen text or image analysis

## Versioning

- v0.1.x — Windows only; OCR, screenshot, template matching, image diff
- v0.2.0 — planned cross-platform image analysis (non-WinRT path)

## License

Apache-2.0
