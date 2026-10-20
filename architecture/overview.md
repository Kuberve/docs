# Обзор архитектуры Kuberve

## Что такое Kuberve

Гибридная операционная система: монолитное ядро с возможностью выноса подсистем в userspace-сервисы.

Цель: единое вычислительное пространство из устройств пользователя — тонкие ARM-клиенты + толстый x86-сервер + облако.

## Компоненты

### Слои

| Слой | Где живёт | Что делает | Знает про |
|---|---|---|---|
| **Hardware** | — | CPU, RAM, UART, GIC, диски, сеть | — |
| **Kernel** | EL1 | Планировщик, MMU, IPC routing, interrupt dispatch, syscall handler | Все устройства (через IRQ dispatch), но не управляет ими |
| **Userspace** | EL0 | Драйверы (uart_server), приложения, клиенты сервисов | Только то, что kernel разрешил через page tables и IPC |

### Kernel — что внутри

| Подсистема | Файл | Ответственность |
|---|---|---|
| Scheduler | `sched.rs` | Потоки, состояния (Ready/Running/Sleeping/Blocked/Dead), приоритеты, aging, sleep, exit, IPC-очереди |
| MMU | `mmu.rs` | L0/L1/L2 page tables, kernel/user/device split, permissions (AP, UXN, PXN) |
| Physical alloc | `phys.rs` | Bitmap 4KB-страниц, `alloc_page` / `free_page` |
| Kernel heap | `allocator.rs` | Bump allocator для `Box`, `Vec`, `format!` в kernel |
| IPC | `ipc.rs` | `Message` (64 байта), `Queue` (8 слотов), ring buffer |
| Interrupts | `gic.rs`, `vectors.rs` | GICv3 init, SPI setup, IRQ handler, EOI, forward к сервисам |
| Timer | `timer.rs` | CNTP_TVAL reload, tick, wake sleepers (только планировщик, не userspace) |
| Syscalls | `syscall.rs`, `vectors.rs` | `svc #0` handler, диспетчеризация по `x8`, сохранение/восстановление контекста (GPR + SIMD) |
| Loader | `loader.rs` | ELF64 парсер, загрузка PT_LOAD в user-регион |

### Userspace — что снаружи

| Процесс | PID | Роль |
|---|---|---|
| `uart_server` | 2 | Драйвер UART: MMIO 0x09000000, IRQ 33, запись и echo |
| `client1`, `client2` | 3, 4 | Тестовые клиенты, шлют `Message{tag=WRITE}` серверу |
| (будущее) `fs_server` | 5+ | ФС, после v0.8 |
| (будущее) `net_server` | 5+ | Сеть, после v0.9 |

### Пример потока: `println!("hello")` в userspace
User (EL0) println!("hello")
→ format! в строку
→ sys_send(uart_pid=2, Message{tag=WRITE, data=[ptr, 6]})
→ svc #0 (trap в EL1)

Kernel (EL1) rust_sync_handler
→ match x8 == SYS_SEND
→ ipc_send_blocking(2, msg)
→ QUEUES[2].push(msg)
→ wake_thread(2) (если uart_server спал)
→ eret (обратно в EL0)

Uart_server sys_recv() → получил Message{tag=WRITE}
(EL0, pid=2) → читает *ptr (identity mapping, видит память client)
→ uart::putc() для каждого байта
→ MMIO write в 0x09000000 (напрямую, без syscall)
→ UART hardware отправляет символ в QEMU terminal

Hardware UART DR → QEMU → stdout терминала

**Что важно:** kernel **не знает** про UART в шаге 3. Он только переслал `Message` из очереди client1 в очередь uart_server. Запись в железо — прямой MMIO из EL0, kernel не участвует.

### Пример потока: ввод символа в QEMU
QEMU Пользователь нажал 'a'
→ UART hardware выставляет RX FIFO not empty
→ IRQ 33 → GIC → CPU

Kernel (EL1) rust_irq_handler
→ intid == 33
→ gic::disable_irq(33) (маскируем, чтобы не спамило)
→ ICC_EOIR1_EL1 (EOI, GIC отпускает)
→ ipc_send(2, Message{tag=IRQ, data=[33]})
→ return (не обрабатывает UART)

Uart_server sys_recv() → Message{tag=IRQ}
(EL0, pid=2) → uart::try_read() читает UART DR = 'a'
→ echo: uart::putc('a')
→ sys_irq_ack(33) (разрешаем IRQ снова)
→ svc #0

Kernel rust_sync_handler
→ SYS_IRQ_ACK → gic::enable_irq(33)
→ eret

Следующий 'b' цикл повторяется

**Что важно:** kernel **не читает** UART DR. Он только пересылает IRQ-событие. Uart_server читает байт, эхо-выводит, разрешает IRQ. Полный interrupt-driven driver в userspace.

### Почему kernel не пустой

Даже в микроядре kernel делает **обязательный минимум**:
- **Планирование** — иначе нет потоков.
- **MMU** — иначе нет изоляции (сейчас identity, но управление всё равно в kernel).
- **IPC routing** — иначе процессы не общаются (нужна очередь в kernel, shared memory — отдельно).
- **Interrupt dispatch** — иначе IRQ не доходят до сервисов (GIC настраивается в kernel).
- **Syscall entry** — иначе EL0 не может позвать kernel.

Это **не** «драйверы в kernel» — это **механизмы**, без которых userspace не работает. Драйверы (UART, FS, network) — вынесены или будут вынесены.

## Kernel — что остаётся

- **Scheduler** — планировщик потоков. States: Ready, Running, Sleeping, Blocked, Dead.
- **MMU** — управление страницами. Kernel в L2[0..1], user в L2[2..4], device в L2_DEV.
- **IPC** — очереди сообщений (8 слотов на поток), blocking send/recv.
- **Interrupt dispatch** — ловит IRQ, для device-IRQ пересылает сервису.
- **Syscall handler** — `svc #0` из EL0/EL1, диспетчеризация по `x8`.

## Userspace — что вынесено

- **UART server** — драйвер UART. Получает MMIO UART, обрабатывает IRQ 33, эхо/запись.
- **Приложения** — клиенты сервисов через IPC.

## MMU — раскладка

| Регион | VA | Права | Что |
|---|---|---|---|
| Device (L2_DEV[72]) | 0x09000000 | AP=01 | UART (доступен из EL0) |
| Kernel code (L2[0]) | 0x40000000 | AP=00 | Только EL1 |
| Kernel data (L2[1]) | 0x40200000 | AP=00 | Только EL1 |
| User code (L2[2]) | 0x40400000 | AP=01 | EL0 exec |
| User data/stack (L2[3]) | 0x40600000 | AP=01 | EL0 RW |
| User heap (L2[4]) | 0x40800000 | AP=01 | EL0 RW |
| Rest (L2[5..512]) | ... | AP=00 | Только EL1 |

## Syscalls

ABI: `x8` = номер, `x0..x5` = аргументы, `x0` = возврат.

Подробности — [spec/syscalls](https://github.com/Kuberve/spec/blob/main/syscalls/README.md).

## IPC

`Message` — 64 байта: `tag: u64`, `data: [u64; 7]`. Очередь на поток — 8 сообщений.

Blocking send: если очередь полна, отправитель блокируется (`Blocked`, `blocked_on_send = target`). При `recv` — будим одного отправителя.

Подробности — [spec/ipc/message.md](https://github.com/Kuberve/spec/blob/main/ipc/message.md).

## Платформы

- **ARM64** — Kuberve Desktop (rk3588s, Orange Pi CM5)
- **x86_64** — Kuberve Hub (после v0.7)