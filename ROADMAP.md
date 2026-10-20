# Kuberve Roadmap

## Продукты

| Продукт | Что | Кому |
|---|---|---|
| **Kuberve Core** | Ядро + ABI + libc | Разработчикам |
| **Kuberve Desktop** | ОС для ARM-клиента | Пользователям |
| **Kuberve Hub** | ОС для x86-сервера (NAS + compute) | Homelab |
| **Kuberve Cloud** | Sync, OTA, VPN, backup, registry | Всем устройствам |
| **Kuberve SDK** | Тулчейн, документация, примеры | Разработчикам приложений |

---

## Kuberve Desktop (ARM64) — вехи

| Веха | Готово | Критерий | Статус |
|---|---|---|---|
| v0.1 Boot | UART, GIC, timer | Tick приходит | ✅ |
| v0.2 Scheduler | states, prio, aging, sleep, exit | Два потока чередуются | ✅ |
| v0.3 Memory | MMU (L2), phys alloc, kernel heap | Box/Vec в kernel | ✅ |
| v0.4 Userspace | EL0, ELF loader, syscalls | Hello from EL0 | ✅ |
| v0.5 IPC | send/recv, blocking, queues | Два процесса обмениваются Message | ✅ |
| v0.6 Drivers in user | UART server, IRQ forwarding | Kernel не трогает UART DR | ✅ |
| **v0.7 Isolation** | per-process AS, copy_to_user, sys_sbrk | A не видит память B | ✅ |
| v0.8 Storage | NVMe driver, VFS, ФС, open/read/write | Программа с диска | ✅ |
| v0.9 Network | TCP/IP, сокеты, DHCP | ping из userspace | ⬜ |
| v0.10 Input | keyboard, mouse, framebuffer, GUI | Окно с текстом | ⬜ |
| v1.0 Desktop | пакетник, init, приложения | Daily driver | ⬜ |

## Kuberve Hub (x86_64) — вехи

| Веха | Готово | Статус |
|---|---|---|
| s0.1 Boot x86 | UEFI, COM1, APIC, HPET | ⬜ |
| s0.2 HAL | arch::aarch64 vs arch::x86_64 | ⬜ |
| s0.3 Microkernel | планировщик, IPC, MMU на x86 | ⬜ |
| s0.4 Userspace x86 | EL0, syscalls, UART server | ⬜ |
| s0.5 Server services | FS, network, control plane | ⬜ |
| s0.6 Cloud | sync, OTA, VPN, DNS, backup | ⬜ |
| s1.0 Hub | Web UI, uptime >6 месяцев | ⬜ |

---

## Когда переходить к x86

**После v0.7 (Isolation), до v0.8 (Storage).**

Причины:
1. ABI стабилизирован (все основные syscalls есть).
2. Понятно, как устроена изоляция → можно абстрагировать в HAL.
3. FS/network — общие для ARM и x86, лучше сразу кроссплатформенно.

**Не раньше v0.7:** каждое изменение ABI придётся делать дважды.
**Не позже v0.8:** иначе FS/network придётся переписывать под x86.

## Параллельные треки

| Трек | Когда |
|---|---|
| Kuberve SDK | после v0.7 |
| Kuberve Cloud | после s0.4 |
| Публичный релиз | после v1.0 |
| Свой carrier board | после v1.0 |
| Свой CM | никогда (если не компания) |

## Что не выносить в userspace

**Timer IRQ 30** — остаётся в kernel. Это тик планировщика, не драйвер. 100 Гц через IPC — не работает. Timer в kernel делают Linux, Redox, QNX.

**Time services** (`clock_gettime`, `alarm`) — syscall в kernel, не отдельный сервер.
