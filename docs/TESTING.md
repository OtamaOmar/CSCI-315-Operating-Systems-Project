# Building and Testing

## Prerequisites

- RISC-V GCC toolchain: `riscv64-unknown-elf-gcc` or `riscv64-linux-gnu-gcc`
- QEMU 7.2 or later: `qemu-system-riscv64`

On Ubuntu:

```sh
sudo apt install gcc-riscv64-unknown-elf qemu-system-misc
```

## Build and boot

```sh
make clean
make qemu
```

xv6 boots to a shell prompt (`$`). Exit QEMU with `Ctrl-A` then `x`.

## Test programs

All test programs are in `user/` and are listed under `UPROGS` in the
`Makefile`. Each prints `PASS:` or `FAIL:` lines.

Several tests run in two steps. The first run writes a checkpoint file and
exits. The second run, with an argument, calls `restore()` on that file.

| Program | Steps | What it checks |
| --- | --- | --- |
| `test_syscall` | `test_syscall`, then `test_syscall restore` | `checkpoint()` returns 0; empty path and wrong pid return -1; `restore()` returns a pid > 0; a missing file returns -1. |
| `test_serial` | `test_serial`, then `test_serial check` | The checkpoint file exists, is not empty, and starts with magic `0x43484B50`. |
| `test_memory` | `test_memory`, then `test_memory verify` | Globals, stack variables, and strings are saved; values are changed after checkpoint; restore succeeds. |
| `test_fd` | `test_fd`, then `test_fd verify` | A file read to offset 4 resumes at offset 4 (`EFG`, then `HIJ`). |
| `test_full` | `test_full`, then `test_full verify` | Integration: memory, file offset, and working directory together. |
| `test_pipe` | `test_pipe` | A self-contained pipe pair checkpoints; a child holding only one end of a shared pipe is rejected with -1. |
| `test_scenario` | `test_scenario` | Checkpoint at `val = 5`, change it, restore, and check the restored value. |

## Troubleshooting

| Error | Fix |
| --- | --- |
| `riscv64-unknown-elf-gcc: command not found` | Install the RISC-V toolchain. |
| `qemu-system-riscv64: command not found` | Install `qemu-system-misc`. |
| `invalid RISC-V ISA string` | Upgrade QEMU to 7.2 or later. |
| Stale behavior after code changes | Run `make clean`. `fs.img` is rebuilt from scratch, which also removes old `.ckpt` files. |
