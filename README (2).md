# Multi-Container Runtime — OS-Jackfruit

A lightweight Linux container runtime written in C, consisting of a long-running **user-space supervisor daemon** (`engine.c`) and a **kernel-space memory monitor** (`monitor.c`). The system supports multi-container lifecycle management, concurrent logging, and real-time memory enforcement via a custom kernel module.

---

## Team

| Name | SRN |
|---|---|
| SHIVANAND| PES2UG24CS473 |
| Shivangouda | PES2UG24CS471 |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   CLI Client (engine)                   │
│  start | run | ps | logs | stop                         │
└────────────────────┬────────────────────────────────────┘
                     │ Unix socket (/tmp/mini_runtime.sock)
┌────────────────────▼────────────────────────────────────┐
│              Supervisor Daemon (engine)                  │
│                                                         │
│  ┌──────────────┐   ┌──────────────┐  ┌──────────────┐ │
│  │  Container   │   │ Bounded Log  │  │   SIGCHLD    │ │
│  │   Registry   │   │   Buffer     │  │   Handler    │ │
│  │  (linked     │   │ (prod/cons   │  │  (reap+      │ │
│  │   list)      │   │  pipeline)   │  │   classify)  │ │
│  └──────┬───────┘   └──────┬───────┘  └──────┬───────┘ │
│         │                  │                  │         │
│  ┌──────▼───────────────────▼──────┐          │         │
│  │   clone() → Container Process  │◄─────────┘         │
│  │   (PID + UTS + mount ns)        │                    │
│  └─────────────────────────────────┘                    │
└──────────────────────────┬──────────────────────────────┘
                           │ ioctl
┌──────────────────────────▼──────────────────────────────┐
│          Kernel Module: container_monitor                │
│                                                         │
│   Linked list of monitored PIDs + limits                │
│   Timer callback (1s) → RSS check → soft/hard enforce  │
│   /dev/container_monitor  ←  REGISTER / UNREGISTER      │
└─────────────────────────────────────────────────────────┘
```

---

## Features

### Task 1 — Supervisor Daemon & Container Isolation
- Long-running supervisor process listening on a Unix domain socket
- Containers launched with `clone()` using `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` flags, providing PID, UTS (hostname), and mount namespace isolation
- Each container gets a private `chroot` filesystem and `/proc` mounted inside
- Scheduling priority configurable via `--nice` (calls `nice()` inside the container before `execl`)

### Task 2 — CLI Interface
- `supervisor <rootfs>` — starts the long-running daemon
- `start <id> <rootfs> <cmd> [opts]` — launches a container in the background
- `run <id> <rootfs> <cmd> [opts]` — launches and blocks until container exits; forwards `SIGINT`/`SIGTERM`
- `ps` — lists all containers with PID, start time, state, and exit info
- `logs <id>` — streams the container's log file to stdout
- `stop <id>` — sends `SIGTERM` to the container, marks it as `stopped`

### Task 3 — Concurrent Logging Pipeline
- **Bounded buffer** (capacity 32 slots, 4 KiB each) with producer/consumer threading
- One **log-reader thread per container** (producer): reads the container's stdout/stderr pipe and pushes chunks into the shared buffer
- One **central logger thread** (consumer): drains the buffer and appends to `logs/<id>.log`
- Graceful shutdown: `bounded_buffer_begin_shutdown()` broadcasts to unblock all waiters; consumer drains remaining items before exiting — no log lines are dropped
- Synchronised with `pthread_mutex` + `pthread_cond` (`not_empty` / `not_full`)

### Task 4 — Kernel Memory Monitor
- Kernel module exposes `/dev/container_monitor`
- Supervisor registers each container PID + soft/hard limits via `ioctl(MONITOR_REGISTER)`
- A kernel timer fires every 1 second; for each registered entry it reads RSS via `get_mm_rss()`
- **Soft limit**: logs a warning once per exceedance; resets if RSS drops back below the threshold
- **Hard limit**: sends `SIGKILL` to the container and removes the entry
- On `SIGCHLD` (container exit), supervisor calls `ioctl(MONITOR_UNREGISTER)` to clean up the entry
- Thread-safe: spinlock (`spin_lock_irqsave`) used throughout, safe for softirq context

### Task 5 — Scheduling Experiments
- `--nice N` flag sets the scheduler priority (range −20 to 19) for each container at launch
- Demonstrated with two simultaneous `cpu_hog` containers: `alpha` at nice=0, `beta` at nice=10
- Both containers observed running concurrently, with alpha completing faster due to higher CPU priority

### Task 6 — Cleanup
- `del_timer_sync` (aliased to `timer_delete_sync` on kernel ≥ 6.15) ensures the timer callback is not running before the list is freed
- Module exit walks the linked list with spinlock held, calling `list_del` + `kfree` on every entry
- Supervisor shutdown: sends `SIGTERM` to all running containers, waits 1 second for graceful exit, then `SIGKILL` stragglers, then reaps all with `waitpid`

---

## Design Decisions

### Spinlock over Mutex in kernel module
The timer callback runs in softirq context. A `mutex_lock` would sleep, which is illegal in softirq context. A spinlock (`spin_lock_irqsave`) is safe here. The trade-off is that `get_rss_bytes()` is called inside the lock; this is acceptable because `get_task_mm` and `get_mm_rss` are safe in this context and the critical section is short.

### `del_timer_sync` → `timer_delete_sync` compatibility
Linux 6.15 renamed `del_timer_sync` to `timer_delete_sync`. A version-guarded `#define` bridges both kernel generations:
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 15, 0)
#define del_timer_sync   timer_delete_sync
#endif
```

### Soft-limit warning reset
`soft_warned` is reset to 0 when RSS drops back below the soft limit. This means the warning fires again on the next exceedance rather than being silenced for the container's lifetime.

### `snprintf` over `strncpy` for cross-boundary copies
`CONTAINER_ID_LEN` (engine, 64 bytes) differs from `MONITOR_NAME_LEN` (kernel, 32 bytes). Using `snprintf(..., sizeof(dest), "%s", src)` avoids compiler truncation warnings and is bounds-safe regardless of source length.

### Bounded buffer capacity
Capacity of 32 log chunks (each 4 KiB = 128 KiB total) was chosen to absorb bursts from multiple containers without blocking producers for extended periods, while keeping memory usage low.

---

## Build & Run

### Prerequisites
```bash
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build
```bash
cd boilerplate
make
```
This produces:
- `engine` — user-space runtime binary
- `monitor.ko` — kernel module
- `memory_hog`, `cpu_hog`, `io_pulse` — test workload binaries

### Set up root filesystems
```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

### Load the kernel module
```bash
sudo insmod monitor.ko
```

### Start the supervisor
```bash
sudo ./engine supervisor ./rootfs-base
```

### Start containers
```bash
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog" --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  "/cpu_hog" --soft-mib 48 --hard-mib 80 --nice 10
```

### Check status
```bash
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Unload the kernel module
```bash
sudo rmmod monitor
```

---

## CLI Options

| Flag | Description | Default |
|---|---|---|
| `--soft-mib N` | Soft memory limit in MiB (warning only) | 40 MiB |
| `--hard-mib N` | Hard memory limit in MiB (SIGKILL) | 64 MiB |
| `--nice N` | Scheduling nice value (−20 to 19) | 0 |

---

## Container States

| State | Meaning |
|---|---|
| `starting` | Container record created, not yet running |
| `running` | Container process active |
| `stopped` | Gracefully stopped via `stop` CLI command |
| `hard_limit_killed` | SIGKILL sent by kernel monitor (RSS exceeded hard limit) |
| `exited` | Container process exited on its own |

---

## Screenshots

### 1. Starting containers, checking ps, viewing logs, stopping

Two containers (`alpha`, `beta`) started simultaneously. `ps` confirms both running. Logs streamed from `alpha`. Both containers stopped cleanly.

![start-ps-logs-stop](screenshots/Screenshot_2026-04-13_213752.png)

### 2. Hard limit enforcement — `hard_limit_killed`

Both `alpha` and `beta` (new run with PIDs 8682/8669) killed by the kernel monitor after exceeding the 80 MiB hard limit. Earlier run shows `stopped` state.

![hard-limit-killed](screenshots/Screenshot_2026-04-13_220543.png)

### 3. Log output from container

`cpu_hog` output correctly captured and streamed via `engine logs alpha`.

![logs](screenshots/Screenshot_2026-04-13_220804.png)

### 4. Kernel module dmesg — register/unregister/process-exit events

`dmesg` output showing the kernel module correctly tracking container registration, process exit detection, and unregister requests across multiple container runs.

![dmesg](screenshots/Screenshot_2026-04-13_222313.png)

### 5. Scheduling experiment — nice 0 vs nice 10

`alpha` (nice=0) and `beta` (nice=10) run `cpu_hog` simultaneously for 20 seconds. Both containers produce output, confirming concurrent scheduling. Alpha has higher CPU priority.

![scheduling](screenshots/Screenshot_2026-04-13_224138.png)

### 6. Cleanup — module unload

`rmmod monitor` fails while a container is still registered (module in use), confirming the reference count is properly held. After container exits, unload succeeds.

![rmmod](screenshots/Screenshot_2026-04-13_224606.png)

---

## File Structure

```
boilerplate/
├── engine.c            # User-space supervisor and CLI
├── monitor.c           # Kernel module (memory monitor)
├── monitor_ioctl.h     # Shared ioctl definitions
├── Makefile            # Builds engine, monitor.ko, test workloads
├── cpu_hog.c           # CPU-bound test workload
├── memory_hog.c        # Memory-consuming test workload
├── io_pulse.c          # I/O-bound test workload
├── environment-check.sh
└── logs/               # Container log files (auto-created)
```

---

## Known Limitations

- The `logs` command returns "Log file not found" if the container has not produced any output yet or if the log directory does not exist before the supervisor starts. Run `mkdir -p logs` before starting the supervisor.
- `sudo rmmod monitor` will fail with "Module is in use" if any container is still registered with the kernel module. Stop all containers first.
- The supervisor must be started as root (for `clone` with namespace flags and `chroot`).

---

## Environment

- **OS**: Ubuntu 24.04 (VMware Virtual Platform)
- **Kernel**: 6.17.0-20-generic
- **Compiler**: GCC 13.3.0
- **Architecture**: x86_64
