# Tauri Desktop Wrapper Review & Suggestions


---

## 1. Security & Permissions (Critical)

### ⚠️ Recommendation: Disable Remote IPC/Tauri Injection if Unused
In `src-tauri/capabilities/default.json`, a `remote` configuration is defined:
```json
"remote": {
  "urls": ["https://billiards.tailuge.workers.dev"]
}
```
* **Analysis**: Under Tauri v2, configuring `remote.urls` instructs Tauri to inject its IPC bindings (`window.__TAURI__`) and expose core capabilities to the remote domain. Since this app is a pure remote wrapper for a hosted Three.js game, **the remote game does not make any Tauri API calls (such as file system, system info, etc.)**.
* **Best Practice**: If the remote site does not interact with Tauri APIs, you should **omit the `remote` block entirely**. By removing it, you prevent the remote origin from executing system-level Tauri commands, drastically reducing the security footprint and protecting the user if the remote domain/CDN is ever compromised.
* **Action**:
  Simply remove the `"remote"` property from `src-tauri/capabilities/default.json`. Standard remote page navigation and WebGL will still work perfectly inside the webview without it.

---

## 2. Rust Build & Binary Size Optimization

Tauri binaries can be large because of standard Rust debug symbol inclusion and default compiler profiles. For a minimal desktop wrapper, keeping the binary size as small as possible is a major benefit.

### ⚡ Recommendation: Configure Size-Optimized Release Profile
Add a production release profile to the bottom of `src-tauri/Cargo.toml` to strip debug symbols, enable Link-Time Optimization (LTO), and compile for size:

```toml
[profile.release]
opt-level = "z"       # Optimize for binary size
lto = true            # Enable Link-Time Optimization
codegen-units = 1     # Reduce parallel codegen units to maximize LTO optimization
panic = "abort"       # Remove panic unwinding overhead
strip = true          # Strip symbols and debug info automatically
```
* **Impact**: Can reduce the output binary size 


---

