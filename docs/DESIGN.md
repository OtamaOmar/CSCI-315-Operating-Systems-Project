# Design

This document describes how checkpoint/restore works in our xv6-riscv kernel.

## System calls

| Call | Number | Kernel entry | Behavior |
| --- | --- | --- | --- |
| `checkpoint(int pid, const char *path)` | 22 | `sys_checkpoint()` | Writes the caller's state to `path`. `pid` must be 0 or the caller's pid. Returns 0, or -1 on error. |
| `restore(const char *path)` | 23 | `sys_restore()` | Builds a new process from `path` and makes it runnable. Returns the new pid, or -1 on error. |

The user stubs are generated from `user/usys.pl`, and the prototypes are in
`user/user.h`. Both entry points live in `kernel/sysproc.c` and call
`checkpoint_proc()` and `restore_proc()` in `kernel/proc.c`.

## File format

```
+---------------------------+
| struct ckpt_header        |
+---------------------------+
| page 0: present byte      |
|         [4096 bytes]      |  only if present == 1
| page 1: present byte      |
|         [4096 bytes]      |
| ...                       |
| page sz/PGSIZE - 1        |
+---------------------------+
```

The header is defined in `kernel/proc.h`:

```c
#define CKPT_MAGIC   0x43484B50   // "CHKP"
#define CKPT_VERSION 1

struct ckpt_fd {
  int   type;      // FD_NONE, FD_PIPE, FD_INODE, FD_DEVICE
  char  readable;
  char  writable;
  uint  inum;      // inode number (FD_INODE, FD_DEVICE)
  uint  off;       // file offset (FD_INODE)
  short major;     // major device number (FD_DEVICE)
  int   pipe_id;   // pairing id (FD_PIPE)
};

struct ckpt_header {
  uint   magic;
  uint   version;
  int    pid;
  uint64 sz;
  struct trapframe tf;
  char   name[16];
  int    num_fds;
  uint   cwd_inum;               // 0 if no cwd
  struct ckpt_fd fds[NOFILE];
};
```

Restore rejects a file whose magic number or version does not match.

## Checkpoint

`checkpoint_proc(p, path)` runs inside one file-system transaction
(`begin_op()` / `end_op()`):

1. Create the output file with `create(path, T_FILE, 0, 0)`.
2. Fill the header: magic, version, pid, `sz`, a copy of the trapframe, and
   the process name.
3. Walk `p->ofile[]`:
   - `FD_INODE`: save inode number, flags, and offset.
   - `FD_DEVICE`: save inode number, flags, and major number.
   - `FD_PIPE`: give each distinct `struct pipe` a pairing id, and count its
     read and write ends.
4. Check pipes. Each pipe must have exactly two FDs in this process: one
   read-only end and one write-only end. Any other pipe shape returns -1.
5. Save the inode number of `p->cwd`.
6. Write the header, then call `checkpoint_addr_space()`.

`checkpoint_addr_space()` (`kernel/vm.c`) walks the page table from 0 to `sz`
one page at a time. For each page it writes a present byte. If the page is
mapped, it writes the page contents straight from the physical address.

The trapframe is copied during the system call. At that point `epc` already
points past the `ecall`, so a restored process resumes at the instruction
after `checkpoint()`.

## Restore

`restore_proc(path)` also runs inside one file-system transaction:

1. Open the file with `namei()`, read the header, and check magic and version.
2. Allocate a process with `allocproc()` and allocate `sz` bytes of user
   memory with `uvmalloc()`.
3. Release `np->lock` before disk I/O, then call `restore_addr_space()`.
   It reads each present byte, then either copies the saved page into the
   new frame or zeroes the frame.
4. Rebuild inode and device FDs. Allocate a `struct file`, set flags, offset
   and major number, and look the inode up again with `iget_by_inum()`.
5. Restore the working directory with `iget_by_inum(cwd_inum)`. If that
   fails, use the root directory.
6. Rebuild pipes. For each pairing id, call `pipealloc()` and put the read
   and write ends back in their original FD slots.
7. Copy the saved trapframe, set `a0 = 1` so the restored `checkpoint()`
   call returns 1, copy the name, set the parent to `initproc`, and mark the
   process `RUNNABLE`.

On any error, restore closes the FDs it opened, drops the cwd reference,
frees the process, and returns -1.

## Helper: `iget_by_inum()`

`kernel/fs.c` adds `iget_by_inum(uint inum)`, a wrapper around
`iget(ROOTDEV, inum)`. It is exported in `kernel/defs.h` so `proc.c` can
turn saved inode numbers back into in-memory inodes.

## Limits

- Self-checkpoint only.
- Pipe buffer contents are not saved. A restored pipe starts empty.
- Only self-contained pipe pairs are supported. A pipe shared with another
  process is rejected at checkpoint time.
- Files are identified by inode number only. Restore does not check that the
  inode still holds the same file.
- The restored process gets a new pid and `init` as its parent.
- Single disk device (`ROOTDEV`) only.
- If checkpoint fails after `create()`, the partial file is left on disk.
