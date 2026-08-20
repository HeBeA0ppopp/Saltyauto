# Saltyy
### Hitreg-accurate autoclicker for Windows  
*Built on Blur AutoClicker's architecture — engine fully rebuilt*

---

## What's different from Blur

### The hitreg bug (and the fix)

**Original Blur behavior:**  
`send_batch()` fires all clicks via a single `SendInput(N, inputs, ...)` call where every `MOUSEINPUT.time` is `0`. The OS timestamps all events identically on dequeue. Games using `WM_INPUT`, `GetRawInputData`, or any input validation that checks for duplicate timestamps see a burst of events all at `t=0` — the dedup logic drops all but the first.

Symptoms: clicks "don't come through," inconsistent hit registration, frame-burst clicks being eaten.

**Saltyy fix:**

1. **Per-click `GetTickCount64()` timestamps.** Every single `MOUSEINPUT` struct gets a fresh timestamp at construction time, not at batch-send time. Even if two clicks land in the same millisecond, the kernel's own scheduler jitter separates them in the input queue.

2. **Individual `SendInput(1, ...)` calls.** Replaced batched `SendInput(N, ...)` with one call per click. The kernel assigns fresh queue positions to each, making them appear as discrete hardware events rather than a software burst.

3. **Hardware-realistic DOWN→UP hold timing.** Physical USB mice report at 125–1000 Hz (1–8ms between report frames). Saltyy draws each click's hold duration from N(8000µs, 2000µs) — an 8ms mean with ±2ms natural variation. Games doing timing analysis on input events see a physical mouse, not a synthetic pulse train.

4. **`NtDelayExecution` for sub-ms inter-click gaps.** `Sleep(1)` has a 15.625ms floor (or 1ms with `timeBeginPeriod(1)`). `NtDelayExecution` with 100-ns tick resolution gives true sub-ms control. Saltyy uses a 500µs inter-click gap — invisible to CPS rate but guarantees `GetTickCount64` advances between consecutive DOWNs.

5. **`GetAsyncKeyState` DOWN verification.** After firing the DOWN event, Saltyy polls the VK state for up to 2ms to confirm the button actually registered in the kernel. If it didn't (focus steal, window switch), it retries once. Games checking VK state mid-frame see a confirmed DOWN, not a phantom event.

6. **`NtSetTimerResolution(5000)`.** 0.5ms resolution (Blur used 10000 = 1ms). Required for the 500µs inter-click gap and the NtDelayExecution hold to actually resolve at sub-ms granularity.

---

## Architecture

```
src-tauri/src/
├── engine/
│   ├── mod.rs          — types, ClickerConfig, SALTYY_EXTRA_INFO tag
│   ├── worker.rs       — main loop, start/stop, config builder, hitreg paths
│   ├── mouse.rs        ← CORE HITREG FILE
│   │                     send_batch_hitreg(), send_single_click_hitreg(),
│   │                     nt_sleep_us(), vk_is_down(), make_input()
│   ├── cycle.rs        — ClickCyclePlan, execute_click_cycle()
│   ├── keyboard.rs     — keyboard press emulation (unchanged)
│   ├── failsafe.rs     — corner/edge stop zones (unchanged)
│   ├── process.rs      — process list / task switcher check (unchanged)
│   ├── rng.rs          — xorshift64 + Box-Muller Gaussian (unchanged)
│   └── stats.rs        — CPS stats logging (unchanged)
├── settings/mod.rs     — settings schema + serde
├── updates/            — update checker
└── lib.rs, hotkeys.rs, icon.rs, overlay.rs, ...
```

### Click dispatch flow

```
run_batch()
  └─ if hitreg_enabled && single cycles:
       send_batch_hitreg(down, up, n, button, inter_gap_us, rng)
         └─ for each click:
              hold_us = rng.next_gaussian(8000, 2000).max(1000)
              send_single_click_hitreg(down, up, hold_us, button)
                ├─ stamp_ms = GetTickCount64() as u32
                ├─ SendInput(1, make_input(down_flag, stamp_ms))
                ├─ poll GetAsyncKeyState(vk) for ≤2ms  [verify DOWN]
                ├─ NtDelayExecution(-hold_us * 10)     [hold]
                ├─ stamp_ms = GetTickCount64() as u32  [fresh stamp for UP]
                └─ SendInput(1, make_input(up_flag, stamp_ms))
              nt_sleep_us(inter_gap_us)  [500µs inter-click gap]
```

### SALTYY_EXTRA_INFO = `0x53A1_77CC`

Every synthetic INPUT has `dwExtraInfo` set to this value. Any `WM_INPUT` hook or `GetRawInputData` consumer can filter our events: `if ri.header.dwExtraInfo == SALTYY_EXTRA_INFO { skip }`.

---

## Building

Requires:
- Rust 1.80+ (`rustup update stable`)
- Node.js 18+ + npm
- Tauri CLI v2 (`cargo install tauri-cli --version "^2"`)
- Windows SDK (comes with VS Build Tools)

```powershell
# install frontend deps
npm install

# dev mode
cargo tauri dev

# release build
cargo tauri build
```

Output: `src-tauri/target/release/Saltyy.exe`  
Installer: `src-tauri/target/release/bundle/nsis/Saltyy_1.0.0_x64-setup.exe`

---

## Hitreg config fields (in ClickerConfig)

| Field | Default | Description |
|-------|---------|-------------|
| `hitreg_enabled` | `true` | Master toggle. Disabling falls back to Blur's original send_batch path. |
| `hitreg_hold_mean_us` | `8000` | Mean DOWN→UP hold in microseconds. 8ms = realistic USB HID cadence. |
| `hitreg_hold_stddev_us` | `2000` | Std-dev of hold duration. 2ms σ gives natural variation. |
| `hitreg_inter_gap_us` | `500` | Gap between clicks (NtDelayExecution). 500µs = invisible to CPS, forces unique GetTickCount64 stamps. |

These are currently hardcoded in `build_config()`. Wire them to the settings schema if you want UI controls.

---

## What wasn't changed

- All UI components (React/TypeScript) — fully preserved, name-patched to Saltyy
- Overlay system
- Hotkey system
- Process list / whitelist / blacklist
- Stop zones / failsafe corners / edge stops
- Click points system
- Smooth mouse movement (cubic bezier)
- Double-click handling (uses the cycle plan executor, not the hitreg batch)
- Stats / logging
- Autostart
- Update checker (endpoint still points at Blur's releases — update `tauri.conf.json`)
