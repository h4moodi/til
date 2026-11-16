# Locking Down Same-UID Memory Access with Yama `ptrace_scope`

*Linux internals & hardening · 2026-11-16*

## What it is

Without extra controls, any process can attach to and read the memory of any other dumpable process running as the same UID. One compromised user-level process can then pull SSH agent keys, browser session tokens or `gpg-agent` secrets out of its neighbours. Yama is a small LSM that adds the `kernel.yama.ptrace_scope` sysctl to narrow who may attach. Its hook runs on every `PTRACE_MODE_ATTACH` access check, so it covers `PTRACE_ATTACH`/`PTRACE_SEIZE` and also `/proc/<pid>/mem` and `process_vm_readv()`.

## Example

```bash
# Check the current mode (most distros default to 1)
sysctl kernel.yama.ptrace_scope

# Allow attach only from processes holding CAP_SYS_PTRACE
echo 'kernel.yama.ptrace_scope = 2' | sudo tee /etc/sysctl.d/10-ptrace.conf
sudo sysctl --system

# Verify: as an unprivileged user this should now fail with EPERM
sleep 600 & gdb -p $! -batch -ex 'info proc'
```

- `0`: classic. Any process with the same UID can attach to a dumpable target.
- `1`: restricted. The tracer must be an ancestor of the target, or the target must name it with `prctl(PR_SET_PTRACER, pid)`. This is why `gdb ./prog` works and `gdb -p` on a sibling process does not.
- `2`: admin-only. Attaching requires `CAP_SYS_PTRACE`.
- `3`: no attach at all, and the setting can't be changed again until reboot.

## When to use it / why it matters

- **Defensive:** use `1` on workstations and `2` on servers and CI runners, where nobody should be debugging live processes. Use `3` only on appliances, because it also breaks `strace -p`, `gdb -p`, some crash reporters and profilers.
- **Offensive (authorized testing):** after you land as a user, `cat /proc/sys/kernel/yama/ptrace_scope` tells you whether memory scraping or injection into same-user processes is possible. A value of `0` is a finding to report.
- **Gotchas:** containers share the host's Yama setting. Granting `CAP_SYS_PTRACE` to a container (`--cap-add SYS_PTRACE`) gets past modes 1 and 2, so audit capabilities alongside the sysctl. `PR_SET_DUMPABLE=0` (used by `ssh-agent`) is a separate per-process defence and works at any Yama level.
- **Detection:** add auditd rules for the `ptrace` and `process_vm_readv` syscalls (for example `-a always,exit -F arch=b64 -S ptrace -k ptrace`) and alert on unexpected tracers.

## Sources

- [Linux kernel docs: Yama LSM](https://docs.kernel.org/7.0/admin-guide/LSM/Yama.html)
- [ptrace(2) man page, "Ptrace access mode checking" and Yama sections](https://man7.org/linux/man-pages/man2/ptrace.2.html)
- [Yama LSM source (`yama_lsm.c`)](https://code.woboq.org/linux/linux/security/yama/yama_lsm.c.html)
