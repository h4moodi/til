# Shrinking Kernel Attack Surface with `kernel.io_uring_disabled`

*Linux internals & hardening · 2026-11-23*

## What it is

io_uring is a fast async I/O interface: userspace submits work through shared ring buffers instead of issuing one syscall per operation. That speed comes with a large, fast-moving kernel codebase that has produced many local privilege escalation bugs, and it also hides activity from tools that only watch syscalls. Since Linux 6.6 (and backported to several distro kernels), the `kernel.io_uring_disabled` sysctl lets you limit or turn off `io_uring_setup()` system-wide without rebuilding the kernel.

## Example

```bash
# Inspect current state
sysctl kernel.io_uring_disabled kernel.io_uring_group

# Allow io_uring only for members of one group (e.g. a DB service account)
sudo groupadd -r iouring && sudo usermod -aG iouring postgres
GID=$(getent group iouring | cut -d: -f3)
printf 'kernel.io_uring_disabled = 1\nkernel.io_uring_group = %s\n' "$GID" \
  | sudo tee /etc/sysctl.d/60-io_uring.conf
sudo sysctl --system
```

- `0`: io_uring available to every process (the default).
- `1`: unprivileged processes can create rings only if they are in `io_uring_group` or hold `CAP_SYS_ADMIN`. The default group is `-1`, which matches nobody.
- `2`: `io_uring_setup()` fails with `EPERM` for everyone, root included. Rings that already exist keep working.

## When to use it / why it matters

- **Defensive:** most servers, CI runners and container hosts never use io_uring. Set `2` on them, or `1` plus a group when a specific service (database, storage, proxy) gains from it. Google cited io_uring as the source of a large share of kCTF exploit submissions and now restricts it on ChromeOS, Android apps and production servers.
- **Detection gap:** ARMO's 2025 "Curing" PoC ran a full C2 agent over io_uring operations (network, file reads and writes) without calling the syscalls that many eBPF/seccomp-based runtime tools hook. Disabling io_uring closes that blind spot. If you can't, monitor `io_uring_setup` calls and the io_uring tracepoints / LSM hooks.
- **Pentest angle (authorized):** `cat /proc/sys/kernel/io_uring_disabled` on a foothold tells you whether that interface is in play for local privesc research or for evasion testing. A value of `0` on a hardened fleet is worth reporting.
- **Gotchas:** check the sysctl exists (`ls /proc/sys/kernel/io_uring_disabled`) before shipping it in config management. Some runtimes (recent Node/libuv, QEMU, some databases) use io_uring if it's there, and most fall back to other I/O paths on `EPERM`. Test before you roll this out to a whole fleet.

## Sources

- [Linux kernel docs: `/proc/sys/kernel/` (io_uring_disabled, io_uring_group)](https://docs.kernel.org/admin-guide/sysctl/kernel.html)
- [LKML: "io_uring: add a sysctl to disable io_uring system-wide" (v4 patch)](https://lkml.iu.edu/hypermail/linux/kernel/2308.2/00994.html)
- [Google Security Blog: Learnings from kCTF VRP's 42 Linux kernel exploits submissions](https://security.googleblog.com/2023/06/learnings-from-kctf-vrps-42-linux.html)
- [ARMO: io_uring rootkit bypasses Linux security tools](https://www.armosec.io/blog/io_uring-rootkit-bypasses-linux-security)
