# Kuberve Roadmap

**Kuberve** is an independent operating system for aarch64 (not Linux, not POSIX). 
The goal is to build a unified computing space: thin ARM clients, a thick x86 hub, 
and cloud services — all working as one ecosystem where the user truly owns their 
hardware and software.

**Status:** pre-alpha. ABI is unstable. Not for production.

---

## 1. Completed Milestones (v0.1 – v0.14.4)

| Milestone | Feature | Status |
|---|---|---|
| **v0.1** | Boot, UART, GICv3, Timer | ✅ |
| **v0.2** | Scheduler (states, priority, aging, sleep) | ✅ |
| **v0.3** | MMU (4-level, ASID), phys alloc, kernel heap | ✅ |
| **v0.4** | EL0, ELF loader, syscalls | ✅ |
| **v0.5** | IPC (`Message`, queues, blocking) | ✅ |
| **v0.6** | Userspace drivers (UART server, IRQ forwarding) | ✅ |
| **v0.7** | Per-thread address spaces (code/stack/heap) | ✅ |
| **v0.8** | NVMe + FAT32 + FD table | ✅ |
| **v0.9** | Network: virtio-net, ARP, IPv4, ICMP, UDP, TCP, DHCP | ✅ |
| **v0.10** | Graphics: virtio-gpu, framebuffer | ✅ |
| **v0.11** | Input: virtio-keyboard, virtio-tablet | ✅ |
| **v0.12** | Shell (`kuberve> `) | ✅ |
| **v0.13** | Package manager (`kv`) | ✅ |
| **v0.14.1** | `sys_spawn`, `sys_wait`, `sys_kill`, `init.cfg` | ✅ |
| **v0.14.2** | Separate ELFs (`/system/bin/init`, `/system/bin/uart`) | ✅ |
| **v0.14.3** | Nested paths, `/apps` installation | ✅ |
| **v0.14.4** | **GUI Rewrite (W.1–W.7)** | ✅ |

### What was achieved in v0.14.4 (GUI Rewrite):
- **W.1:** Double-buffer compositor (`FB_BACK` → `present_damage` → `FB`).
- **W.2:** Per-window bitmaps (`[u32; 320*240]` in `.bss`), `draw_chrome`, `blit_to_back`.
- **W.3:** Z-order (`z_order: [usize]`), focus, click-to-raise.
- **W.4:** Drag-and-drop without artifacts (cursor hide/show).
- **W.5:** `fill_rect`, `draw_text_in_win`, dynamic window background.
- **W.6:** `SYS_GUI_CREATE_WINDOW` (30), userspace API for creating windows from EL0.
- **W.7:** Keyboard routing to focused window, `put_char` with dynamic backspace.

---

## 2. Current Sprint (v0.14.5 – v0.15.x)

These are the immediate blockers and the next feature set.

| Task | Type | Status |
|---|---|---|
| ~~Fix `virtio-net` TX timeout~~ | Bugfix | ✅ v0.14.5 |
| **Cleanup warnings** (dead code, static_mut_refs) | Chore | 🚧 |
| **`kv update`** (local repo parsing) | Feature | ⬜ |
| **`kv search`** (local metadata) | Feature | ⬜ |
| **HTTP client** (for remote repositories) | Feature | ⬜ |
| **Ed25519 signatures for `.kvpkg`** | Security | ⬜ |

*Note on the TX timeout:* After transitioning to `opt-level = 2`, the virtio-net TX 
queue sometimes times out during high system load (e.g., simultaneous GUI rendering 
and network). This causes DHCP and TCP SYN packets to fail. Requires adding proper 
`wfe`/`dsb` synchronization in the virtio ring polling loop.

---

## 3. Phase 2: Package Manager & HTTP (v0.15.5 – v0.15.7)

| Milestone | Description |
|---|---|
| **v0.15.5** | `kv update` — connect to a remote HTTP repository and update the package index. |
| **v0.15.6** | `kv install` from HTTP — download `.kvpkg` over TCP, verify hash, extract to `/apps`. |
| **v0.15.7** | `kv remove` with dependency resolution (post-order tree removal). |

*Target:* A user can run `kv install hello` on a fresh Kuberve system and fetch it 
from the internet.

---

## 4. Phase 3: KuberveFS (v0.16.x)

Replacing FAT32 with a native file system.

| Milestone | Description |
|---|---|
| **v0.16.1** | **KuberveFS Design:** Inode-based, COW (Copy-on-Write) snapshots. |
| **v0.16.2** | **Mount points, symlinks, hardlinks.** |
| **v0.16.3** | **Capabilities:** Drop uid/gid. Use capability-based access control (e.g., `CAP_WRITE`). |
| **v0.16.4** | **Snapshots:** Rollback the system state to a previous COW point. |

*Why:* FAT32 is 8.3 and has no permissions. KuberveFS will support long Unicode 
names, case sensitivity, and atomic snapshots.

---

## 5. Phase 4: Production Network (v0.17.x)

Current TCP implementation is minimal (basic handshake, no retransmit).

| Milestone | Description |
|---|---|
| **v0.17.1** | **TCP Retransmit:** RTO calculation, exponential backoff. |
| **v0.17.2** | **Window Scaling (RFC 1323).** |
| **v0.17.3** | **Congestion Control:** Basic Reno or CUBIC. |
| **v0.17.4** | **`curl`-like utility** in userspace. |

---

## 6. Phase 5: Full Desktop (v0.18.x)

Moving from a WM to a desktop environment.

| Milestone | Description |
|---|---|
| **v0.18.1** | **Compose:** Layers, shadows, alpha-blending in the compositor. |
| **v0.18.2** | **Widgets:** Buttons, text inputs, checkboxes, scrollbars. |
| **v0.18.3** | **File Manager:** A GUI app written in userspace. |
| **v0.18.4** | **Text Editor:** Simple editor with syntax highlighting. |
| **v0.18.5** | **Terminal Emulator:** A GUI terminal (replacing the raw console). |
| **v0.18.6** | **Settings:** Themes, wallpaper, font sizes. |

---

## 7. Phase 6: Real Hardware (v0.19.x)

Porting from QEMU to the Orange Pi CM5 / RK3588S.

| Milestone | Description |
|---|---|
| **v0.19.1** | **Native Drivers:** UART, GPIO, Power Management on RK3588S. |
| **v0.19.2** | **Boot on CM5:** U-Boot, DTB, kernel image loading. |
| **v0.19.3** | **Native Input:** USB keyboard, HDMI display. |

---

## 8. Phase 7: x86_64 Hub (v0.20.x)

Transitioning to the server ecosystem.

| Milestone | Description |
|---|---|
| **v0.20.1** | **Boot x86:** Multiboot2 or UEFI, COM1 serial. |
| **v0.20.2** | **HAL:** Abstract `arch::aarch64` vs `arch::x86_64` (MMU, IRQ, Timer). |
| **v0.20.3** | **Microkernel on x86:** Scheduler, IPC, MMU, Syscalls. |
| **v0.20.4** | **Userspace x86:** Port `init`, `uart`, `kv` to x86. |
| **v0.20.5** | **Server Services:** File server, network server, control plane. |

---

## 9. Phase 8: Cloud & Ecosystem (v0.21.x – v0.22.x)

| Milestone | Description |
|---|---|
| **v0.21.x** | **Web UI** for Kuberve Hub (remote management). |
| **v0.22.x** | **Sync, OTA, VPN, Backup** services. |
| **v0.23.x** | **SDK Public Release** (tutorials, examples, WASM runtime). |

---

## 10. v1.0 (The Daily Driver)

| Milestone | Description |
|---|---|
| **v1.0.0** | **Kuberve Desktop:** Stable daily driver on CM5. |
| **v1.0 Hub** | **Uptime > 6 months** on x86. |
| **v1.0 Ecosystem** | Desktop + Hub + Cloud working together. |

---

## Architecture Principles (Non-Negotiable)

1. **Not POSIX.** Custom ABI (`x8`, `svc #0`). No `fork()`, `read()`, `write()`.
2. **Microkernel-inspired, but pragmatic.** Drivers in userspace. Scheduler, MMU, IPC in kernel. Timer remains in kernel (QNX-style).
3. **WASM + WASI for apps.** No custom programming language. No custom compiler.
4. **Open Source with Commercial Protection.** AGPLv3 for kernel/user, LGPLv3 for libc, MIT for ABI. Commercial licensing available for closed-source products. Trademark "Kuberve" is protected.
5. **Solo-dev scalable.** One logical change per PR. No feature creep.

---

## What We Don't Do

- **No custom programming language.** Use WASM + WASI. Write apps in Rust, C, Zig, etc.
- **No custom compiler.** Use LLVM.
- **No POSIX compatibility layer.** It's a different model, not "Linux but different".
- **No x86 before ARM is stable.** (We start x86 at v0.20, after real hardware bring-up).
- **No proprietary locks.** Everything core is Open Source.

---

## How to Contribute

Check the [CONTRIBUTING.md](https://github.com/Kuberve/kernel/blob/main/CONTRIBUTING.md) 
for details on CLA, coding style, and language policy.

Key areas needing help right now:
- **Network stack:** fixing TX timeouts, TCP retransmits.
- **KuberveFS:** designing the inode table, COW.
- **Real hardware:** RK3588S bring-up.
- **English translations:** inline comments in old modules.

---

*Last updated: v0.14.4 release.*