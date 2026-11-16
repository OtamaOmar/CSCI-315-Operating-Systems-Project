# xv6 Process Checkpoint/Restore

Course project for CSCI 315 Operating Systems.

We added process checkpoint/restore to xv6-riscv. A running process calls
`checkpoint()` to write its state to a file. Later, any process calls
`restore()` on that file, and the kernel rebuilds the process: user memory,
registers, open file descriptors (with offsets), working directory, and
self-contained pipes. The restored process resumes right after its
`checkpoint()` call, as a new child of `init`.

## Team

| Member | Contribution |
| --- | --- |
| Omar Tarek Amer | Syscall plumbing and user API (`SYS_checkpoint`, `SYS_restore`, `user.h`, `usys.pl`), refactor of the checkpoint/restore path, tests, integration |
| Abdelrahman Bassem | First checkpoint/restore implementation and initial test program |
| Kamel Walid Ramadan | Checkpoint file format (serialization) and address-space save/restore |
| Hamza Salah | File descriptors, working directory, and pipes |
| Aly | Test suite (`test_syscall`, `test_serial`, `test_memory`, `test_fd`, `test_pipe`, `test_full`, `test_scenario`) |

## Build and run

Requires a RISC-V toolchain (`riscv64-unknown-elf-gcc` or
`riscv64-linux-gnu-gcc`) and `qemu-system-riscv64`.

```sh
make qemu
```

At the xv6 shell, run the test programs. Most tests run in two steps: the
first run writes a checkpoint, the second run restores it.

```sh
$ test_syscall          # then: test_syscall restore
$ test_serial           # then: test_serial check
$ test_memory           # then: test_memory verify
$ test_fd               # then: test_fd verify
$ test_full             # then: test_full verify
$ test_pipe
$ test_scenario
```

Exit QEMU with `Ctrl-A` then `x`. See [docs/TESTING.md](docs/TESTING.md) for
what each test checks.

## API

```c
int checkpoint(int pid, const char *path);  // pid 0 = calling process
int restore(const char *path);              // returns new pid, or -1
```

`checkpoint()` returns 0 to the caller and -1 on error. In the restored
process, the same call returns 1.

## Design overview

A checkpoint file is a fixed header followed by the address space.

- **Header:** magic number, format version, original pid, memory size,
  trapframe (all user registers and the program counter), process name,
  the file descriptor table, and the inode number of the working directory.
- **Memory:** for each page from 0 to `sz`, one "present" byte, followed by
  the page contents if the page is mapped.
- **File descriptors:** inode and device FDs are saved by inode number,
  flags, offset, and major device number. Restore looks each inode up again
  by number.
- **Working directory:** saved by inode number. Restore falls back to `/`
  if it is missing.
- **Pipes:** each pipe gets a pairing id. Restore creates a new pipe for each
  read/write pair and puts both ends back in their original FD slots.

Kernel code: `checkpoint_proc()` and `restore_proc()` in `kernel/proc.c`,
`checkpoint_addr_space()` and `restore_addr_space()` in `kernel/vm.c`,
`iget_by_inum()` in `kernel/fs.c`, and the syscall entry points in
`kernel/sysproc.c`. Details are in [docs/DESIGN.md](docs/DESIGN.md).

## Known limits

- A process can only checkpoint itself (`pid` must be 0 or its own pid).
- Pipe buffer contents are not saved. A restored pipe starts empty.
- Only pipes whose read end and write end are both held by the process, once
  each, are supported. A process holding one end of a shared pipe gets -1.
- Files are restored by inode number. If a file is deleted or its inode is
  reused between checkpoint and restore, the restored FD points at the wrong
  inode or restore fails.
- The restored process gets a new pid and `init` as its parent.
- Single disk device (`ROOTDEV`) only.

## License

Based on MIT's [xv6-riscv](https://github.com/mit-pdos/xv6-riscv). The
original xv6 README is kept in [README.xv6](README.xv6), and the license in
[LICENSE](LICENSE).
