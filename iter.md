# Tauri Desktop Wrapper Review & Suggestions

This review evaluates the Tauri Billiards Wrapper repository against best practices for Rust and Tauri v2.
Since the project is intended to be **extremely minimal**, the focus of this review is on:
1. **Security** (safeguarding system access when loading remote content).
2. **Performance & Binary Size Optimization** (maximizing Rust/Tauri efficiency).
3. **CI/CD Efficiency** (speeding up GitHub Actions run times).
4. **Native Desktop UX** (making the web app feel like a polished native app).

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

### 🔒 Recommendation: Explicit Content Security Policy (CSP)
Currently, `"csp": null` is set under `app.security` in `tauri.conf.json`.
* **Analysis**: Unset CSP fallback to open webview defaults, which allows the remote webview to load scripts, styles, or iframes from any origin.
* **Best Practice**: Restrict loaded resources only to trusted origins (the game workers domain, CDN, and local WebGL resources).
* **Action**:
  Update `tauri.conf.json` with a secure, game-compatible CSP:
  ```json
  "security": {
    "csp": "default-src 'self' https://billiards.tailuge.workers.dev; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://billiards.tailuge.workers.dev; style-src 'self' 'unsafe-inline' https://billiards.tailuge.workers.dev; img-src 'self' data: https://billiards.tailuge.workers.dev; connect-src 'self' https://billiards.tailuge.workers.dev"
  }
  ```

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
* **Impact**: Can reduce the output binary size by **50% to 70%** (saving tens of megabytes on Linux, Windows, and macOS) and improve load-up times.

---

## 3. CI/CD Pipeline (`build.yml`) Improvements

### 🚀 Recommendation: Use `cargo-binstall` to Avoid Compiling `tauri-cli`
Currently, the GitHub Actions runner installs the Tauri CLI via:
```yaml
- name: Install Tauri CLI
  run: cargo install tauri-cli --version "^2"
```
* **Analysis**: `cargo install` downloads and compiles `tauri-cli` and all its dependencies from source on every single run across all matrix OS platforms. This typically wastes **3 to 6 minutes** of CI time per runner!
* **Best Practice**: Use `cargo-binstall` to download pre-compiled Tauri CLI binaries, which takes only 5–10 seconds.
* **Action**:
  Update that step in `.github/workflows/build.yml` to:
  ```yaml
  - name: Install Tauri CLI
    run: |
      curl -L --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/cargo-bins/cargo-binstall/main/install-from-binstall-release.sh | bash
      cargo binstall -y tauri-cli@2
  ```

---

## 4. Native Desktop UX & Polish

To elevate the app from "a browser wrapper" to a "native game desktop application," consider these low-code/config tweaks:

### 🎮 Recommendation: Disable Context Menus & DevTools in Production
By default, users can right-click the game to see a browser context menu, or potentially press `F12` or `Ctrl+Shift+I` to open inspector panels. For a game, this breaks immersion.
* **Action**: Update `src-tauri/src/lib.rs` to disable the default context menu and developer tools in release builds:

```rust
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    let mut builder = tauri::Builder::default();

    // Disable devtools in production release builds
    #[cfg(not(debug_assertions))]
    {
        builder = builder.setup(|app| {
            use tauri::Manager;
            if let Some(window) = app.get_webview_window("main") {
                // Disable right-click context menu via webview configuration
                // or inline JavaScript evaluation to prevent native menu popups.
                let _ = window.eval(r#"
                    document.addEventListener('contextmenu', e => e.preventDefault());
                "#);
            }
            Ok(())
        });
    }

    builder
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### ⌨️ Recommendation: Intercept Standard Reload/Zoom Shortcuts
Prevent players from accidentally refreshing the game page (via `F5` or `Ctrl+R`) or zooming (via `Ctrl++` / `Ctrl+-`), which might ruin active game sessions.
* **Action**: Since this is a minimal app, you can easily intercept keyboard events via a minimal custom JavaScript script evaluated inside the webview at startup.
  ```rust
  let _ = window.eval(r#"
      document.addEventListener('keydown', e => {
          // Disable F5, Ctrl+R, CMD+R, and zoom keys
          if (
              e.key === 'F5' ||
              ((e.ctrlKey || e.metaKey) && e.key === 'r') ||
              ((e.ctrlKey || e.metaKey) && (e.key === '=' || e.key === '-' || e.key === '0'))
          ) {
              e.preventDefault();
          }
      });
  "#);
  ```

### 💾 Recommendation: Save and Restore Window State (Optional Plugin)
If a user resizes or moves the window, they expect it to open in the same position next time. Tauri provides an official, minimal, zero-boilerplate plugin for this.
* **Implementation**:
  1. Add `tauri-plugin-window-state` to dependencies.
  2. Register it in `src-tauri/src/lib.rs`:
     ```rust
     builder = builder.plugin(tauri_plugin_window_state::Builder::default().build());
     ```
  Since this is an optional enhancement, we keep it as a suggestion depending on how "minimal" you want the core codebase dependencies to remain.

---

## Summary Checklist for Next Iteration

| Item | Recommendation | Priority | Complexity |
|---|---|---|---|
| **Security** | Omit `"remote"` block from default capabilities | **High** | Very Low |
| **Security** | Set secure, strict Content Security Policy (CSP) | **Medium** | Low |
| **Rust** | Configure size-optimized release profiles in `Cargo.toml` | **High** | Very Low |
| **CI/CD** | Use `cargo-binstall` in GitHub Actions build | **High** | Low |
| **UX** | Evaluate inline JS to prevent right-click menus & refreshes | **Medium** | Low |
