# Multi-Container Runtime — OS Jackfruit

## 1. Team Information

| Name | SRN |
|---|---|
| Kshitij G Shettigar | PES1UG24CS240 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build

```bash
cd boilerplate
make
```

This builds: `engine`, `monitor.ko`, `memory_hog`, `cpu_hog`, `io_pulse`.

### Prepare rootfs

```bash
mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

### Load kernel module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

### Start supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

### Launch containers (in a second terminal)

```bash
sudo ./engine start alpha ./rootfs-alpha /cpu_hog
sudo ./engine start beta ./rootfs-beta /cpu_hog
```

### List containers

```bash
sudo ./engine ps
```

### Inspect logs

```bash
sudo ./engine logs alpha
```

### Stop containers

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Run memory limit test

```bash
cp memory_hog rootfs-alpha/
sudo ./engine start alpha ./rootfs-alpha /memory_hog --soft-mib 5 --hard-mib 10
sudo dmesg | grep container_monitor | tail -10
```

### Run scheduler experiments

```bash
# Experiment 1: different priorities
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --nice -5
sudo ./engine start beta ./rootfs-beta /cpu_hog --nice 10
sleep 12
cat logs/alpha.log
cat logs/beta.log

# Experiment 2: CPU-bound vs I/O-bound
sudo ./engine start alpha ./rootfs-alpha /cpu_hog
sudo ./engine start beta ./rootfs-beta /io_pulse
sleep 12
cat logs/alpha.log
cat logs/beta.log
```

### Unload module and clean up

```bash
# Stop supervisor with Ctrl+C first, then:
sudo rmmod monitor
```

---

## 3. Demo Screenshots

### Screenshot 1 — Multi-container supervision
Two containers (alpha and beta) launched and running under one supervisor process.
<img width="1920" height="1003" alt="triple_terminal" src="https://github.com/user-attachments/assets/8b2d7a0c-a9cb-41c7-8b2f-9d3efa669973" />

### Screenshot 2 - Scheduler Experiment 1 — CPU priority comparison
<img width="1920" height="1003" alt="2nd_pic" src="https://github.com/user-attachments/assets/70cddec8-6343-4b65-88ce-4e814a5b15e0" />

### Screenshot 3 — Bounded-buffer logging
Log file contents captured through the producer-consumer logging pipeline, showing cpu_hog output routed from container stdout through the bounded buffer to persistent log files.
<img width="1920" height="1003" alt="another_one" src="https://github.com/user-attachments/assets/906ef4c9-1eac-42fe-b85b-6ba60691a5df" />

### Screenshot 4 — CLI and IPC
`engine stop` and `engine ps` commands issued over the UNIX domain socket control channel, supervisor responding with updated container state.
<img width="1920" height="1003" alt="Experiment_2" src="https://github.com/user-attachments/assets/1a7915b3-e318-4258-8200-21f912ec46ea" />

### Screenshot 5 — Soft-limit warning & Hard-limit enforcement
`dmesg` output showing `SOFT LIMIT` warning event when container RSS exceeded the soft limit threshold.
`dmesg` output showing `HARD LIMIT` kill event when container RSS exceeded the hard limit, container terminated by the kernel module.
<img width="1920" height="1003" alt="Memory_limits" src="https://github.com/user-attachments/assets/736ce558-79c2-41dc-bb7a-cd8f048709f3" />

### Screenshot 6 — Scheduling experiment
Log output from two containers running with different nice values (-5 vs 10), showing higher accumulator values for the high-priority container in the same time window.
<img width="1105" height="975" alt="Exp_2_terminal3" src="https://github.com/user-attachments/assets/0a72c88e-7504-4caf-999a-ad2b8a057e73" />

### Screenshot 7 — Clean teardown
All containers stopped, no stray processes in `ps aux`, kernel module unloaded cleanly with `Module unloaded.` in dmesg.
<img width="918" height="513" alt="Final_terminal2_teardown_ss" src="https://github.com/user-attachments/assets/fc311bcb-4cc8-4d0d-9950-1b5120af7834" />

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Linux namespaces are kernel features that partition global system resources so each container sees its own isolated view. This project uses three namespace types via the `clone()` syscall with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS`.

**PID namespace (`CLONE_NEWPID`):** The container process becomes PID 1 in its own PID namespace. Processes inside cannot see or signal host processes. The host kernel maintains a mapping between container PIDs and host PIDs — the supervisor uses host PIDs for monitoring and signaling.

**UTS namespace (`CLONE_NEWUTS`):** Each container gets its own hostname, set via `sethostname()` inside `child_fn`. This prevents containers from reading or modifying the host's hostname.

**Mount namespace (`CLONE_NEWNS`):** The container gets its own mount table. We call `chroot()` into the Alpine rootfs and mount `/proc` inside it. This gives the container a completely isolated filesystem view — it cannot see the host filesystem.

What the host kernel still shares with all containers: the network stack (no `CLONE_NEWNET`), the host's IPC namespace, and the underlying hardware. All containers share the same kernel — there is no separate kernel per container as in a VM. The host kernel enforces isolation through namespace bookkeeping in kernel data structures.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is useful because container management requires persistent state. If the CLI itself launched containers directly, there would be no central place to track metadata, route logs, or enforce limits across multiple concurrent containers.

**Process creation:** The supervisor calls `clone()` with namespace flags instead of `fork()`. This atomically creates the child in new namespaces. The child runs `child_fn()` which sets up the container environment and calls `execv()` to replace itself with the requested command.

**Parent-child relationships:** The supervisor is the parent of all container processes. When a container exits, the kernel sends `SIGCHLD` to the supervisor. Without `waitpid()`, the child becomes a zombie — its exit status is held in the kernel's process table until the parent reaps it.

**Metadata tracking:** The supervisor maintains a linked list of `container_record_t` structs, one per container. Each record stores the host PID, start time, state, memory limits, and log path. This list is protected by a mutex since multiple threads (logger thread, signal handler, CLI accept loop) may access it concurrently.

**Signal delivery:** `SIGTERM` sent to a container is delivered via `kill(host_pid, SIGTERM)` from the supervisor. The supervisor itself handles `SIGINT`/`SIGTERM` for orderly shutdown — it signals all containers, joins the logger thread, and closes file descriptors before exiting.

### 4.3 IPC, Threads, and Synchronization

This project uses two IPC mechanisms:

**Pipes (logging):** Each container's stdout and stderr are redirected into a pipe via `dup2()` before `execv()`. The supervisor holds the read end. A per-container producer thread reads from this pipe and pushes chunks into the bounded buffer. This is file-descriptor-based IPC — data flows from container process to supervisor through the kernel's pipe buffer.

**UNIX domain socket (control plane):** CLI commands use a UNIX domain socket at `/tmp/mini_runtime.sock`. The supervisor binds and listens; each CLI invocation connects, sends a `control_request_t`, receives a `control_response_t`, and exits. This is a separate IPC path from logging, as required.

**Bounded buffer synchronization:**

The bounded buffer uses a mutex and two condition variables (`not_full`, `not_empty`).

Without synchronization, the following race conditions exist:
- A producer and consumer could simultaneously read/modify `head`, `tail`, and `count`, corrupting the buffer state.
- A producer could write to a full buffer, overwriting unread data.
- A consumer could read from an empty buffer, returning garbage.

The mutex ensures only one thread modifies the buffer at a time. `not_full` makes producers wait when the buffer is full rather than busy-waiting or dropping data. `not_empty` makes the consumer wait when the buffer is empty rather than spinning. A spinlock would be inappropriate here because the wait periods can be long and spinning wastes CPU.

**Metadata list synchronization:** The `container_record_t` linked list is protected by a separate `metadata_lock` mutex. This is kept separate from the buffer lock to avoid holding both locks simultaneously, which would risk deadlock.

### 4.4 Memory Management and Enforcement

**What RSS measures:** Resident Set Size is the amount of physical RAM currently occupied by a process — pages that are actually loaded in memory, not swapped out. RSS does not measure virtual memory (allocated but not yet accessed), memory-mapped files that are not yet faulted in, or shared library pages counted once per library regardless of how many processes use them.

**Why soft and hard limits are different policies:** A soft limit is a warning threshold — it signals that memory usage is growing and allows the runtime to log, alert, or take preparatory action without disrupting the workload. A hard limit is an enforcement threshold — the process is killed because it has consumed more memory than the system can safely allow. Having two levels gives operators visibility before a container is terminated.

**Why enforcement belongs in kernel space:** A user-space monitor can be delayed by the scheduler — if the monitoring process is not scheduled for several seconds, a runaway container could consume all available RAM before the user-space monitor gets CPU time. A kernel timer fires reliably every second regardless of user-space scheduling. Additionally, the kernel has direct access to `mm_struct` RSS counters and can send signals atomically without race conditions between checking and acting.

### 4.5 Scheduling Behavior

**Experiment 1 — Different nice values:**
Alpha (nice=-5) consistently produced higher accumulator values than beta (nice=10) over the same 10-second window. Linux's Completely Fair Scheduler assigns virtual runtime weights based on nice values — a lower nice value gives a process a larger share of CPU time. Alpha received more CPU time per scheduling period, allowing it to complete more arithmetic iterations.

**Experiment 2 — CPU-bound vs I/O-bound:**
The cpu_hog container ran continuously without blocking, while io_pulse spent most of its time sleeping between I/O iterations. The CFS scheduler naturally gives more CPU time to the cpu_hog because io_pulse voluntarily yields the CPU during sleep. I/O-bound processes accumulate less virtual runtime and are scheduled with higher priority when they wake up (to improve responsiveness), but they still consume far less total CPU than a CPU-bound process running at the same nice value.

These results confirm that Linux scheduling is work-conserving — idle CPU time is never wasted, and CPU-bound processes receive all available cycles that I/O-bound processes voluntarily give up.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** Used `clone()` with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` and `chroot()`.
**Tradeoff:** `chroot()` is simpler than `pivot_root()` but does not fully unmount the old root — a privileged process could escape. `pivot_root()` would be more secure.
**Justification:** For a lab environment, `chroot()` provides sufficient isolation and is significantly simpler to implement correctly.

### Supervisor Architecture
**Choice:** Single long-running supervisor process with a sequential accept loop.
**Tradeoff:** The accept loop handles one CLI request at a time. A concurrent supervisor with a thread per connection would be more responsive under load.
**Justification:** Container operations (start, stop, ps) are infrequent and fast. Sequential handling avoids complex concurrent access to the metadata list.

### IPC and Logging
**Choice:** Pipes for logging, UNIX domain socket for control plane.
**Tradeoff:** Opening and closing the log file on every buffer pop adds I/O overhead. Keeping log files open persistently would be faster but requires tracking open file descriptors per container.
**Justification:** Log writes are infrequent enough that per-write open/close is acceptable, and it avoids file descriptor leaks on abnormal container exit.

### Kernel Monitor
**Choice:** Mutex to protect the monitored list.
**Tradeoff:** A mutex can sleep, which is safe in ioctl context (process context) but would be illegal in interrupt context. A spinlock would work in both contexts but wastes CPU during contention.
**Justification:** Both the timer callback and ioctl handler run in process or softirq context where sleeping is permitted. The mutex is the correct choice here.

### Scheduler Experiments
**Choice:** Used `nice` values via the `--nice` flag passed through the container config.
**Tradeoff:** `nice` affects the CFS weight but does not provide hard CPU guarantees. Cgroups would give precise CPU bandwidth control but require additional kernel setup.
**Justification:** `nice` values are sufficient to demonstrate observable scheduling differences and are directly supported by the existing container infrastructure.

---

## 6. Scheduler Experiment Results

### Experiment 1 — CPU-bound containers with different priorities

| Container | Nice Value | Final Accumulator | Duration |
|---|---|---|---|
| alpha | -5 (high priority) | 1462114265827574355 | 10s |
| beta | +10 (low priority) | 1307381733407785632 | 10s |

**Analysis:** Alpha computed approximately 11.8% more work than beta in the same time window. The CFS scheduler assigned alpha a larger CPU share due to its lower nice value. The difference would be more pronounced on a single-core system — on multi-core, both containers can run simultaneously on different cores, reducing the observable priority effect.

### Experiment 2 — CPU-bound vs I/O-bound containers

| Container | Workload | CPU Usage | Iterations in 12s |
|---|---|---|---|
| alpha | cpu_hog | Continuous | 10 full accumulator loops |
| beta | io_pulse | Bursty (sleeps between I/O) | ~20 I/O iterations |

**Analysis:** The cpu_hog ran at 100% CPU utilization continuously. The io_pulse process spent most of its time sleeping, voluntarily yielding the CPU. The CFS scheduler gave all available CPU time to cpu_hog while io_pulse was sleeping, and quickly scheduled io_pulse when it woke up for its next I/O burst. This demonstrates the scheduler's work-conserving and responsiveness properties — CPU-bound work gets maximum throughput while I/O-bound work gets low latency on wakeup.
