# Exactly-Once Syscalls Across Crashes (xv6-riscv)

**Course:** CSL3030 Operating Systems Lab, IIT Jodhpur  
**Project:** Project 4 — Exactly-once syscalls across crashes  
**Scope:** 3-Student 60% Vertical Slice Implementation  
**Base:** [mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv) (original xv6 documentation in `README`)

---

## Overview

In distributed systems and agentic workflows, retrying an unacknowledged file operation after a crash or network timeout often leads to duplicate writes. 

This project implements an **exactly-once execution guarantee** for file modifications across system crashes in xv6:
1. Operations carry a 64-bit idempotency key via the `effect(key, op, path, buf, n)` system call.
2. The kernel maintains an on-disk persistent table (`key -> result`).
3. The file modification (`APPEND`) and the table insertion are executed inside the **same xv6 log transaction**, guaranteeing crash atomicity.
4. A host-side Python agent drives xv6 inside QEMU via a framed serial bridge, logging requests to a durable journal before dispatch.

---

## Quick Start

### Prerequisites (Ubuntu / WSL2)
```bash
sudo apt update
sudo apt install -y gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu qemu-system-misc python3 make
```

### Build & Run
```bash
# Build kernel and disk image
make clean
make -j8 kernel/kernel fs.img

# Interactive xv6 shell in QEMU
make qemu

# End-to-end crash recovery demonstration
python3 demo.py --crash post

# Run test suite
./run_tests.sh
```

---

## Project Status (60% Scope)

### Completed
- **Full 64-bit Idempotency Key**: Safely read via `argaddr` without 32-bit truncation.
- **On-Disk Persistent Table**: Reserved disk blocks configured at `mkfs`, tracked in the superblock.
- **Atomic `APPEND`**: Bounded append to existing regular files committed in the same transaction as the table record.
- **Replay Protection**: Duplicate keys return stored results with zero additional writes.
- **Concurrency Serialization**: Global sleeplock acquired prior to `begin_op()` preventing races.
- **Crash Injection & Recovery**: Test hooks (`crashat` / `crashhook`) triggering pre-commit and post-commit panics; recovery verified across reboots on the same disk image.
- **Host Agent & Bridge**: Python driver with schema validation, durable pre-send journal, and QEMU serial framing.
- **Prompt Injection Defense**: Tool-layer allowlist refusing prohibited actions from untrusted guest content.

### Deferred for Future Milestones
- **`CREATE` Syscall Operation**: Focus was prioritized on rock-solid atomicity for `APPEND`.
- **Exhaustive Crash-Point Sweep**: Tested representative pre-commit and post-commit points; intermediate write sweep deferred.
- **Fine-Grained Concurrency**: Global sleeplock used instead of per-key bucketed locks.
- **Kernel-Level Taint Enforcement**: Security refusal enforced at host validator layer.
- **Dynamic LLM Integration**: Rule-based deterministic planner used for reproducible testing.

---

## Directory Layout

```text
├── agent/            # Host-side Python agent, planner, validator, and QEMU bridge
├── docs/             # Technical design notes, interface contracts, and crash documentation
├── evidence/         # Raw QEMU logs and verification transcripts
├── fixtures/         # Input data files for guest disk image
├── kernel/           # xv6 kernel: effect syscall, persistent table, crash hooks
├── mkfs/             # Filesystem formatting tool reserving table blocks
├── tests/            # Test harness and independent disk image verifier
├── user/             # xv6 user utilities: effectctl, crashctl, effstat, agentcall
├── demo.py           # Narrated crash/recovery demo script
├── Makefile          # Kernel & userland build rules
└── run_tests.sh      # Automated test runner script
```
