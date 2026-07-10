---
title: Learning Plan
nav_order: 2
---

# Operating Systems Learning Plan

A bottom-up curriculum on how Linux actually works: from the syscall boundary up
through processes, scheduling, memory, concurrency, IPC, filesystems, linking,
boot, kernel modules, containers, and tracing. Every lesson has a lab on your
Linux VM using `strace`, `/proc`, `gdb`, and small C programs — you will *watch*
the kernel do each thing, not just read about it.

This track assumes the networking track's Phase 1 (namespaces) mindset and
cross-links the virtualization track where they meet (hugepages, NUMA, sVirt,
booting a kernel in QEMU).

{: .note }
> Before Lesson 1, prepare your VM with the [Lab Setup]({{ '/lab-setup.html' | relative_url }})
> page — this track additionally needs `build-essential`, `gdb`, and `ltrace`.
> Work the lessons in order — each builds on the ones before it.

---

## [Phase 1: The Kernel Boundary](lessons/phase-01-kernel-boundary.html)

### [Lesson 1: What an operating system actually does](lessons/lesson-01-what-an-os-does.html)
**Goal:** Build the core mental model: the kernel is a privileged program that owns the hardware and rents it out.
**Topics:** Kernel vs user space; privilege (rings, cross-link virt Lesson 04); what the kernel owns (CPU time, memory, devices, files); policy vs mechanism; monolithic vs microkernel.
**Checkpoint:** Why can't a user program simply read another program's memory?

### [Lesson 2: System calls — the only door in](lessons/lesson-02-syscalls-strace.html)
**Goal:** See that *everything* a program does to the outside world is a syscall.
**Topics:** What a syscall is; `strace` basics; reading `strace` output for `cat`, `ls`; syscall numbers; the syscall table; error returns and `errno`.
**Checkpoint:** `printf` is not a syscall. What syscall does it eventually make, and when?

### [Lesson 3: libc, the ABI, and the vDSO](lessons/lesson-03-libc-abi-vdso.html)
**Goal:** Understand the layer between your code and the kernel.
**Topics:** glibc as syscall wrapper + much more; the ABI contract; `ltrace` vs `strace`; the vDSO (why `gettimeofday` is "free"); static vs dynamic glibc.
**Checkpoint:** Why does `clock_gettime` often not appear in strace output?

### [Lesson 4: /proc and /sys — the kernel's dashboard](lessons/lesson-04-proc-sys.html)
**Goal:** Master the two virtual filesystems you'll use in every later lab.
**Topics:** `/proc/PID/*` tour (status, maps, fd, stack); `/proc` system-wide files; `/sys` device tree; these are kernel data structures rendered as text.
**Checkpoint:** `/proc/PID/fd` shows symlinks. To what, and why is that useful for debugging?

### [Lesson 5: Interrupts and the timer tick](lessons/lesson-05-interrupts-timers.html)
**Goal:** Understand how the kernel gets control of the CPU back.
**Topics:** Hardware interrupts vs exceptions vs syscalls; `/proc/interrupts`; softirqs (`/proc/softirqs`); the timer tick and tickless kernels; jiffies and high-resolution timers.
**Checkpoint:** A program runs an infinite loop with interrupts as the only kernel entry. How does the kernel ever schedule someone else?

---

## [Phase 2: Processes](lessons/phase-02-processes.html)

### [Lesson 6: Anatomy of a process](lessons/lesson-06-process-anatomy.html)
**Goal:** Know what a process *is* in kernel terms.
**Topics:** task_struct conceptually; PID/PPID/TGID; the process tree (`pstree`); `/proc/PID/status` field-by-field; threads share, processes don't.
**Checkpoint:** What is the difference between a PID and a TGID?

### [Lesson 7: fork and exec](lessons/lesson-07-fork-exec.html)
**Goal:** Understand the two-step process creation model — and why it's two steps.
**Topics:** `fork()` (copy) then `execve()` (replace); copy-on-write; what survives exec (fds!) and what doesn't; write and strace a mini-shell fork/exec in C; `vfork`/`posix_spawn` briefly.
**Checkpoint:** After `fork()`, both processes think they got the return value. How do they tell each other apart?

### [Lesson 8: Process lifecycle — zombies, orphans, wait](lessons/lesson-08-process-lifecycle.html)
**Goal:** Follow a process from birth to reaping.
**Topics:** States (R/S/D/T/Z) in `ps`; why zombies exist (exit status must be collected); `wait()`/`waitpid()`; orphans reparented to init; the uninterruptible-sleep trap (D state).
**Checkpoint:** Can you kill a zombie with `kill -9`? Why or why not?

### [Lesson 9: Signals](lessons/lesson-09-signals.html)
**Goal:** Understand the kernel's asynchronous notification mechanism.
**Topics:** What a signal delivery actually is; default actions; handlers and `sigaction`; blocked/pending masks in `/proc/PID/status`; why SIGKILL/SIGSTOP can't be caught; signal-safety.
**Checkpoint:** A process ignores SIGTERM. What are your escalation options and their costs?

### [Lesson 10: Process groups, sessions, and job control](lessons/lesson-10-sessions-job-control.html)
**Goal:** Explain what actually happens on Ctrl-C, and how daemons detach.
**Topics:** Sessions, process groups, the controlling terminal; foreground/background; `Ctrl-C`/`Ctrl-Z` signal routing; `nohup`, `setsid`, `disown`; how a daemon daemonizes.
**Checkpoint:** Why does Ctrl-C kill your pipeline but not your shell?

### [Lesson 11: Credentials and capabilities](lessons/lesson-11-credentials-capabilities.html)
**Goal:** Understand who a process *is* and what it may do.
**Topics:** Real/effective/saved UID; setuid binaries; how `sudo` and `passwd` work; capabilities as root split into pieces (`CAP_NET_ADMIN` — you used it all through networking); `getcap`/`setcap`.
**Checkpoint:** `ping` works without sudo on your VM. Find the two possible reasons and check which applies.

---

## [Phase 3: Scheduling](lessons/phase-03-scheduling.html)

### [Lesson 12: Context switches](lessons/lesson-12-context-switches.html)
**Goal:** Know what switching tasks costs and how to see it.
**Topics:** What must be saved/restored; voluntary vs involuntary switches (`/proc/PID/status`); measuring with `pidstat -w`; why context switches hurt (cache, TLB); syscall vs context switch.
**Checkpoint:** Which workload context-switches more — a busy web server or a video encoder? Why?

### [Lesson 13: The scheduler — CFS to EEVDF](lessons/lesson-13-cfs-eevdf.html)
**Goal:** Understand "fair" scheduling well enough to predict behavior.
**Topics:** Runqueues per CPU; vruntime and fairness; the EEVDF successor; timeslices are dynamic; `/proc/sched_debug` and `/proc/PID/sched`; watch two CPU hogs share a core.
**Checkpoint:** Two CPU-bound tasks on one core — what share does each get? What if one sleeps half the time?

### [Lesson 14: Priorities — nice and real-time](lessons/lesson-14-priorities-realtime.html)
**Goal:** Use the priority knobs and know when they matter.
**Topics:** nice values and their (weak) effect; `chrt` and SCHED_FIFO/RR; priority inversion; why real-time on a busy box is dangerous; SCHED_IDLE/BATCH.
**Checkpoint:** When does `nice 19` actually make a visible difference — always, or only under contention?

### [Lesson 15: Affinity, load, and pressure](lessons/lesson-15-affinity-psi.html)
**Goal:** Read system load correctly and place work deliberately.
**Topics:** `taskset` and CPU affinity (cross-link virt CPU pinning); what load average really counts (D state included!); PSI (`/proc/pressure/*`) as the modern signal; per-CPU stats in `mpstat`.
**Checkpoint:** Load average is 8 on a 4-core box but CPU usage is 5%. What's going on and which file confirms it?

---

## [Phase 4: Memory](lessons/phase-04-memory.html)

### [Lesson 16: Virtual memory — the grand illusion](lessons/lesson-16-virtual-memory.html)
**Goal:** Internalize that every address your program sees is fake.
**Topics:** Virtual vs physical addresses; why VM exists (isolation, overcommit, sharing); pages (4 KiB); the MMU; same virtual address, different contents in two processes — prove it in C.
**Checkpoint:** Two processes both print a pointer value `0x5555...`. Same memory? How do you know?

### [Lesson 17: Page tables and the TLB](lessons/lesson-17-page-tables-tlb.html)
**Goal:** Understand the translation machinery (foundation for virt EPT lesson).
**Topics:** Multi-level page tables; page table entries (present, writable, NX); the TLB as translation cache; TLB flushes on context switch; `perf stat` TLB miss counters.
**Checkpoint:** Why does a 4-level walk not make every memory access 5× slower in practice?

### [Lesson 18: A process's memory layout](lessons/lesson-18-process-memory-layout.html)
**Goal:** Read `/proc/PID/maps` fluently.
**Topics:** Text/data/heap/stack/mmap regions; ASLR; `brk` vs `mmap` allocation (watch malloc choose); stack growth and guard pages; RSS vs VSZ.
**Checkpoint:** malloc(1 GiB) succeeds instantly but RSS barely moves. Explain.

### [Lesson 19: Page faults and demand paging](lessons/lesson-19-page-faults.html)
**Goal:** See that memory is allocated by *faulting*, not by malloc.
**Topics:** Minor vs major faults; demand paging; copy-on-write faults after fork (measure them); `ps -o min_flt,maj_flt`; page fault costs.
**Checkpoint:** Which causes a major fault: first touch of malloc'd memory, or first read of an mmap'd file after drop_caches?

### [Lesson 20: The page cache](lessons/lesson-20-page-cache.html)
**Goal:** Understand why "free" memory isn't free and files are fast the second time.
**Topics:** Read/write caching; `free -h` buff/cache; dirty pages and writeback; `sync`; `/proc/meminfo` tour; drop_caches for experiments; `vmtouch`-style verification with timing.
**Checkpoint:** Reading a 1 GiB file twice: second read is 50× faster. Where did the data come from, and what could evict it?

### [Lesson 21: Swap and memory reclaim](lessons/lesson-21-swap-reclaim.html)
**Goal:** Know what the kernel does under memory pressure — before the OOM killer.
**Topics:** Reclaim: clean page cache first, then swap anonymous pages; swappiness; watching `vmstat` si/so; zswap/zram; why some swap usage is healthy.
**Checkpoint:** Is "the box is swapping" always a problem? What number distinguishes healthy from thrashing?

### [Lesson 22: Overcommit and the OOM killer](lessons/lesson-22-oom-overcommit.html)
**Goal:** Understand who dies when memory runs out, and how to influence it.
**Topics:** Overcommit policies; the OOM score (`/proc/PID/oom_score`, `oom_score_adj`); reading an OOM kill in `dmesg`; trigger one safely in a cgroup; cgroup memory limits as containment.
**Checkpoint:** Your database got OOM-killed instead of the leaking script. Which knob prevents a repeat, and what's the trade-off?

### [Lesson 23: Hugepages and NUMA](lessons/lesson-23-hugepages-numa.html)
**Goal:** Connect memory hardware topology to performance (bridges to virt Lessons 19/21/51).
**Topics:** Why 2 MiB pages help (TLB reach); transparent vs explicit hugepages; NUMA nodes and `numactl`; local vs remote access cost; when THP hurts.
**Checkpoint:** Which benefits more from hugepages: random access over 40 GiB, or sequential streaming? Why?

---

## [Phase 5: Concurrency](lessons/phase-05-concurrency.html)

### [Lesson 24: Threads vs processes](lessons/lesson-24-threads.html)
**Goal:** Know exactly what threads share and what that buys/costs.
**Topics:** `clone()` flags — threads and processes are the same thing with different sharing; pthreads; `/proc/PID/task/`; thread stacks; when to prefer threads vs processes.
**Checkpoint:** One thread calls `chdir()`. Does it affect its siblings? What about `errno`?

### [Lesson 25: Race conditions — build one, watch it fail](lessons/lesson-25-race-conditions.html)
**Goal:** Feel why unsynchronized shared mutation is broken, not just know it.
**Topics:** A counter incremented by two threads (write it, run it, lose updates); why `x++` is three operations; data races vs race conditions; ThreadSanitizer.
**Checkpoint:** The racy counter is *sometimes* correct. Why does that make races worse, not better?

### [Lesson 26: Mutexes and futexes](lessons/lesson-26-mutex-futex.html)
**Goal:** Understand what a lock actually is, down to the syscall.
**Topics:** Fix Lesson 25's race with a mutex; the futex — uncontended locks cost zero syscalls (prove with strace); spinlocks vs sleeping locks; lock contention costs.
**Checkpoint:** strace shows no futex calls though the code locks a mutex constantly. Is the mutex working?

### [Lesson 27: Condition variables and semaphores](lessons/lesson-27-condvars-semaphores.html)
**Goal:** Coordinate threads that must *wait for something to happen*.
**Topics:** The producer/consumer problem; condvars and the wait/check loop (spurious wakeups); semaphores; build a tiny bounded queue in C.
**Checkpoint:** Why must `pthread_cond_wait` be called in a `while` loop, not an `if`?

### [Lesson 28: Deadlock](lessons/lesson-28-deadlock.html)
**Goal:** Create, diagnose, and prevent deadlocks.
**Topics:** The four conditions; write a two-lock deadlock; diagnosing with gdb thread apply all bt; lock ordering as the practical cure; timeouts and trylock.
**Checkpoint:** Your program hangs. How do you distinguish deadlock from livelock from starvation using gdb and top?

### [Lesson 29: Atomics and memory ordering](lessons/lesson-29-atomics-ordering.html)
**Goal:** Understand lock-free primitives and why the hardware reorders your code.
**Topics:** Atomic operations (`__atomic_*`); compare-and-swap; memory ordering (relaxed/acquire/release) at mental-model level; why the racy counter is fixable with one atomic; when locks are still better.
**Checkpoint:** An atomic counter fixed the race. Why can't you build a bank-transfer (two variables) with two separate atomics?

### [Lesson 30: Async and thread-pool patterns](lessons/lesson-30-async-patterns.html)
**Goal:** Map high-level concurrency patterns to the primitives underneath.
**Topics:** Thread pools; work queues; event loops vs threads (preview of Phase 8); async/await in your language of choice maps to these; choosing a model for I/O-bound vs CPU-bound work.
**Checkpoint:** A service handles 10k mostly-idle connections. Threads-per-connection or event loop? Justify with numbers from this phase.

---

## [Phase 6: Inter-Process Communication](lessons/phase-06-ipc.html)

### [Lesson 31: Pipes and FIFOs](lessons/lesson-31-pipes-fifos.html)
**Goal:** Understand the mechanism behind every `|` you've ever typed.
**Topics:** `pipe()` and fd inheritance across fork; how the shell wires `a | b`; pipe buffers and backpressure (blocking writes); SIGPIPE; named FIFOs.
**Checkpoint:** `yes | head -1` exits immediately though `yes` runs forever. What killed `yes`?

### [Lesson 32: Unix domain sockets](lessons/lesson-32-unix-sockets.html)
**Goal:** Master the workhorse of local IPC (Docker, systemd, X11 all use it).
**Topics:** Stream/dgram Unix sockets; socket files and permissions as access control; passing file descriptors between processes (SCM_RIGHTS!); `ss -x`; compare to TCP on localhost.
**Checkpoint:** Why can a Unix socket pass an open file descriptor when a TCP socket can't?

### [Lesson 33: Shared memory](lessons/lesson-33-shared-memory.html)
**Goal:** Use the fastest IPC and understand its synchronization price.
**Topics:** `shm_open` + `mmap`; POSIX vs SysV shm; two processes sharing a counter (need Phase 5 tools!); `/dev/shm`; who uses it (databases, virtiofs — cross-link virt Lesson 30).
**Checkpoint:** Shared memory is the fastest IPC. What did you give up compared to a pipe?

### [Lesson 34: Message queues and D-Bus](lessons/lesson-34-message-queues.html)
**Goal:** Know the message-oriented options and where the desktop/system bus fits.
**Topics:** POSIX mqueues (priorities, persistence); when messages beat streams; D-Bus in one lab (`busctl`); signals vs methods vs properties.
**Checkpoint:** What does a message queue give you that a pipe doesn't, and what does D-Bus add on top?

### [Lesson 35: eventfd, signalfd, timerfd](lessons/lesson-35-eventfd-signalfd.html)
**Goal:** See how modern programs turn *everything* into a file descriptor.
**Topics:** eventfd for wakeups; signalfd (signals without handlers); timerfd; why "everything is an fd" matters — it feeds epoll (Phase 8); pidfd.
**Checkpoint:** Why would a program prefer signalfd over a signal handler for SIGCHLD?

---

## [Phase 7: Files & Filesystems](lessons/phase-07-files.html)

### [Lesson 36: File descriptors, dup, and redirection](lessons/lesson-36-file-descriptors.html)
**Goal:** Understand the fd table well enough to explain every shell redirection.
**Topics:** fd table → open file description → inode (three layers, shared offsets!); `dup2` and how `2>&1` works (order matters); O_CLOEXEC; `/proc/PID/fd` and `lsof`.
**Checkpoint:** `cmd 2>&1 >file` and `cmd >file 2>&1` differ. Trace both through the fd table.

### [Lesson 37: VFS and inodes](lessons/lesson-37-vfs-inodes.html)
**Goal:** Understand the abstraction that makes ext4, /proc, and tmpfs all "files".
**Topics:** The VFS layer; inodes vs filenames; hard links vs symlinks (`stat`, link counts); directories are files mapping names→inodes; dentry cache.
**Checkpoint:** Deleting a file that a process still has open frees no space. Explain via inode link/refcounts, and how `lsof` finds it.

### [Lesson 38: ext4 and journaling](lessons/lesson-38-ext4-journaling.html)
**Goal:** Know what's on disk and why a crash doesn't corrupt the filesystem.
**Topics:** Superblock, block groups, extents; `dumpe2fs`, `debugfs` exploration on a loop device; the journal and ordered mode; fsck; what "corruption" journaling does and doesn't prevent.
**Checkpoint:** Journaling protects filesystem metadata. Does it protect the contents of your half-written file? What does?

### [Lesson 39: The filesystem zoo — xfs, btrfs, tmpfs, overlayfs](lessons/lesson-39-filesystem-zoo.html)
**Goal:** Choose filesystems deliberately and understand overlayfs before the container phase.
**Topics:** xfs vs ext4; btrfs CoW & snapshots (compare qcow2, virt Lesson 23); tmpfs is the page cache with a namespace; overlayfs lower/upper/work — build one by hand.
**Checkpoint:** In your hand-built overlay, you delete a file that lives in the lower layer. What actually happens on disk?

### [Lesson 40: Mounts and bind mounts](lessons/lesson-40-mounts-bind.html)
**Goal:** Master the mount table — the container filesystem primitive.
**Topics:** `findmnt`; mount options (ro, noexec, nosuid); bind mounts; mount propagation (private/shared/slave); mount namespaces preview; `pivot_root` conceptually.
**Checkpoint:** Why does `mount -o remount,ro` on a bind mount not affect the original mount point?

### [Lesson 41: Buffered vs direct I/O, and fsync](lessons/lesson-41-io-fsync.html)
**Goal:** Know when your data is *actually* on disk (bridges virt Lesson 25 caching modes).
**Topics:** The write path: write() → page cache → writeback → disk; `fsync`/`fdatasync`/`O_SYNC`; O_DIRECT; what databases do; measure the cost of fsync-per-write.
**Checkpoint:** `write()` returned success and the machine lost power. Is your data safe? List every layer that could still hold it.

### [Lesson 42: The block layer and I/O schedulers](lessons/lesson-42-block-layer.html)
**Goal:** Follow a write below the filesystem.
**Topics:** bio/request queues; I/O schedulers (mq-deadline, bfq, none) and when each; `iostat -x` fields that matter (await, util); `blktrace` peek; readahead.
**Checkpoint:** `%util` is 100% but the NVMe disk isn't saturated. Why is this metric misleading on modern SSDs?

---

## [Phase 8: Event-Driven & Async I/O](lessons/phase-08-async-io.html)

### [Lesson 43: epoll and non-blocking I/O](lessons/lesson-43-epoll.html)
**Goal:** Understand the mechanism behind every modern server and event loop.
**Topics:** Blocking vs non-blocking fds; readiness model; `select`→`poll`→`epoll` evolution; level vs edge triggered; write a tiny epoll echo server in C; C10K in hindsight.
**Checkpoint:** epoll says a socket is readable; your read gets EAGAIN anyway. How is that possible and why must your code tolerate it?

### [Lesson 44: io_uring](lessons/lesson-44-io-uring.html)
**Goal:** See the completion model that's replacing readiness for high-performance I/O.
**Topics:** Submission/completion rings; readiness vs completion; why file I/O never worked with epoll; syscall batching; a small liburing lab; security history in brief.
**Checkpoint:** epoll tells you "you may now read without blocking"; io_uring tells you something different. What, and why does that matter for disk files?

### [Lesson 45: Zero-copy — sendfile and splice](lessons/lesson-45-zero-copy.html)
**Goal:** Eliminate the copies you didn't know you were paying for.
**Topics:** The four copies in read()+write(); `sendfile`, `splice`, `vmsplice`; measure the difference serving a large file; where nginx/kafka use this; mmap as alternative.
**Checkpoint:** Count the user↔kernel copies: cp via read/write loop vs sendfile. Where did they go?

---

## [Phase 9: Linking & Loading](lessons/phase-09-linking.html)

### [Lesson 46: ELF and static linking](lessons/lesson-46-elf-static-linking.html)
**Goal:** Know what's inside a binary.
**Topics:** ELF sections and segments (`readelf`); symbols (`nm`); the linker's job (relocation); compile hello.c with -v and watch the stages; static binaries.
**Checkpoint:** What's the difference between a section and a segment, and which does the kernel care about at exec time?

### [Lesson 47: The dynamic loader](lessons/lesson-47-dynamic-loader.html)
**Goal:** Understand what happens between execve and main().
**Topics:** The interpreter (`ld-linux.so`); `ldd`; library search order (rpath, LD_LIBRARY_PATH, cache); lazy binding and the PLT/GOT; `LD_DEBUG=libs` trace.
**Checkpoint:** `execve` succeeded but the program printed "error while loading shared libraries". Who printed that — the kernel, libc, or something else?

### [Lesson 48: Shared libraries and versioning](lessons/lesson-48-shared-libraries.html)
**Goal:** Build, version, and debug shared libraries.
**Topics:** Build a .so; soname/real name/linker name; symbol visibility; ABI compatibility and why glibc rarely breaks; `dlopen` plugins.
**Checkpoint:** Why do library files have three names (libfoo.so → libfoo.so.1 → libfoo.so.1.2.3), and who uses each?

### [Lesson 49: LD_PRELOAD and symbol interposition](lessons/lesson-49-ld-preload.html)
**Goal:** Intercept any library call — the userspace superpower.
**Topics:** Symbol resolution order; write an LD_PRELOAD shim that logs every `open()`; `dlsym(RTLD_NEXT)`; legitimate uses (fakeroot, jemalloc, torify) and why setuid ignores it.
**Checkpoint:** Your preload shim doesn't fire for a static binary or for raw syscalls. Explain both gaps.

---

## [Phase 10: Boot & Init](lessons/phase-10-boot.html)

### [Lesson 50: From firmware to kernel](lessons/lesson-50-boot-process.html)
**Goal:** Follow the machine from power-on to the first kernel message (pairs with virt Lesson 14).
**Topics:** UEFI → bootloader (GRUB/systemd-boot) → kernel + initramfs; kernel command line (`/proc/cmdline`); early printk; watch it all in a QEMU serial console.
**Checkpoint:** Where does the kernel command line come from, and name two parameters you've already used in other tracks.

### [Lesson 51: initramfs](lessons/lesson-51-initramfs.html)
**Goal:** Understand the tiny userspace that mounts your real root.
**Topics:** Why it exists (modules needed to find root); unpack yours (`lsinitramfs`/cpio); what /init does; switch_root; build a minimal initramfs with one shell — boot it in QEMU.
**Checkpoint:** Your root is on LVM-on-LUKS. Why is booting without an initramfs essentially impossible?

### [Lesson 52: systemd — PID 1](lessons/lesson-52-systemd.html)
**Goal:** Understand init's job and read systemd fluently.
**Topics:** PID 1's special role (reaping, supervision); units/targets/dependencies; `systemctl` + `journalctl` workflow; writing a unit for your own script; systemd is also cgroup manager (preview Phase 12).
**Checkpoint:** What two jobs must PID 1 do even in the most minimal system, and what happens if it dies?

### [Lesson 53: Kernel modules and udev](lessons/lesson-53-modules-udev.html)
**Goal:** Understand how the kernel grows at runtime and devices appear in /dev.
**Topics:** `lsmod`/`modprobe`/`modinfo`; dependencies; how device nodes appear (uevents → udev); writing a udev rule; module parameters (you used kvm's in virt Lesson 09).
**Checkpoint:** You plug in a USB disk and /dev/sdb appears. List the chain of events from interrupt to device node.

---

## [Phase 11: Kernel Interfaces & Security](lessons/phase-11-kernel-security.html)

### [Lesson 54: Write a kernel module](lessons/lesson-54-kernel-module.html)
**Goal:** Cross the boundary — run your own code in ring 0.
**Topics:** hello-world module (init/exit, printk); the build system (Kbuild); insmod/rmmod; why the kernel API is unstable; oops = you crashed the kernel; do it in a VM you can throw away.
**Checkpoint:** Your module has a bug that dereferences NULL. What happens to the system, compared to the same bug in a user program?

### [Lesson 55: A character device](lessons/lesson-55-char-device.html)
**Goal:** Implement `/dev/yourdevice` and finally see "everything is a file" from the other side.
**Topics:** Major/minor numbers; file_operations (open/read/write); a device that returns a message; misc devices; how /dev/null and /dev/urandom are implemented.
**Checkpoint:** When your program reads /dev/yourdevice, whose code executes — the reader's, the kernel's, or your module's? In what context?

### [Lesson 56: seccomp](lessons/lesson-56-seccomp.html)
**Goal:** Shrink the syscall attack surface of a process (pairs with virt Lesson 57 QEMU sandboxing).
**Topics:** seccomp strict vs filter mode; BPF filters (cross-link networking eBPF phase); write a C program that sandboxes itself; how Docker/systemd/Chrome use it; `SystemCallFilter=` lab.
**Checkpoint:** seccomp blocks syscalls by number. Why is blocking `open` but allowing `openat` a useless filter?

### [Lesson 57: LSM — AppArmor and SELinux](lessons/lesson-57-lsm.html)
**Goal:** Understand mandatory access control (completes the sVirt story from virt Lesson 56).
**Topics:** DAC vs MAC; LSM hooks; AppArmor profiles (read one, confine a script); SELinux contexts/type enforcement at reading level; where each distro landed.
**Checkpoint:** Root can read every file under DAC. Under MAC, can it? What does that buy for a compromised daemon running as root?

### [Lesson 58: Build and boot your own kernel](lessons/lesson-58-build-kernel.html)
**Goal:** Demystify the kernel itself — configure, build, boot (capstone).
**Topics:** Getting the source; `make menuconfig` (what to keep); building; booting your kernel in QEMU with your Lesson 51 initramfs; bisecting as a debugging superpower (concept).
**Checkpoint:** Your custom kernel boots but has no network card in QEMU. Describe your debugging path (config? module? cmdline?).

---

## [Phase 12: Containers & Resource Control](lessons/phase-12-containers.html)

### [Lesson 59: Namespaces — all of them](lessons/lesson-59-namespaces.html)
**Goal:** Generalize what you learned in networking Lesson 01 to every namespace type.
**Topics:** mnt/pid/net/uts/ipc/user/cgroup/time namespaces; `unshare` and `nsenter` labs for each; PID namespace quirks (PID 1 inside); user namespaces and uid mapping — rootless containers.
**Checkpoint:** Inside a new PID namespace, `ps` still shows all host processes. What did you forget, and why does that fix it?

### [Lesson 60: cgroups v2](lessons/lesson-60-cgroups.html)
**Goal:** Meter and limit CPU, memory, and I/O for process groups.
**Topics:** The unified hierarchy; cpu.max, memory.max/high, io.max; build limits by hand in /sys/fs/cgroup; watch throttling happen; systemd slices; PSI per cgroup.
**Checkpoint:** memory.max vs memory.high — one OOM-kills, one throttles. Which is which, and when do you want each?

### [Lesson 61: Container images and overlayfs](lessons/lesson-61-container-images.html)
**Goal:** See that an image is just tarballs + a manifest, layered by overlayfs (Lesson 39 pays off).
**Topics:** Pull an image with skopeo/crane and unpack it; layers, manifests, content addressing; how Docker assembles overlay mounts (`docker inspect` the GraphDriver); image vs container filesystem.
**Checkpoint:** Ten containers run from one 1 GiB image. Roughly how much disk do they use before writing anything, and why?

### [Lesson 62: Build a container by hand — capstone](lessons/lesson-62-build-a-container.html)
**Goal:** Combine namespaces + cgroups + pivot_root + seccomp into a working container, no Docker.
**Topics:** A shell script (or small C program) that: unshares namespaces, sets up an overlayfs root, pivot_roots, mounts /proc, applies a cgroup and drops capabilities — then runs a shell. Compare with what runc actually does.
**Checkpoint:** List everything your hand-rolled container is still missing compared to `docker run` (there's a real list — make it).

---

## [Phase 13: Tracing & Debugging](lessons/phase-13-tracing.html)

### [Lesson 63: perf](lessons/lesson-63-perf.html)
**Goal:** Answer "where is the CPU time going?" with evidence.
**Topics:** Sampling vs counting; `perf stat` (IPC, cache misses, from Lesson 17); `perf top`/`record`/`report`; flame graphs; kernel vs user time attribution.
**Checkpoint:** perf shows 40% of time in `_raw_spin_lock`. What class of problem is this, and which earlier lesson's tools confirm it?

### [Lesson 64: ftrace and kprobes](lessons/lesson-64-ftrace-kprobes.html)
**Goal:** Trace *any* kernel function as it runs.
**Topics:** tracefs; function/function_graph tracers; tracepoints vs kprobes; `trace-cmd`; watch the syscall path from Lesson 2 actually execute; relationship to the eBPF tools you know.
**Checkpoint:** You want to know which process is calling `unlink` on your files. Build the exact one-liner (tracepoint or kprobe — justify).

### [Lesson 65: Core dumps and crash analysis](lessons/lesson-65-core-dumps.html)
**Goal:** Do a post-mortem on a dead process.
**Topics:** ulimit/core_pattern/coredumpctl; open a core in gdb (bt, frames, variables); debugging symbols and -g; analyze a provided crashing program; the same idea for kernels (kdump, conceptually).
**Checkpoint:** The backtrace shows `??` for every frame. Two likely causes and the fix for each?

### [Lesson 66: Debugging methodology — capstone](lessons/lesson-66-debugging-capstone.html)
**Goal:** Combine the whole track: a structured hunt through a misbehaving system.
**Topics:** The USE method; a layered toolbox map (top/PSI → perf → strace/ltrace → ftrace → gdb); three staged mystery problems (CPU, memory leak, blocked-on-I/O) to solve with a written diagnosis each.
**Checkpoint:** Write your personal one-page runbook: symptom → first three commands → decision tree. (Compare with the model.)

---

*All 66 lessons written = track complete. Update CLAUDE.md's index as each lands.*
