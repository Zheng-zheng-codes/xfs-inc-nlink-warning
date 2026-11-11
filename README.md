# WARNING in inc_nlink (fs/inode.c:465) via XFS mkdir (v7.4-rc2)

Syzkaller-found bug on Linux v7.4-rc2, reproduced and verified on the same
kernel build (the fuzzing pipeline's reproducer pipeline confirmed the
reproducer; an external standalone chunked-replay run on a fresh VM also
hit — see "External verification").

## Summary

Creating a directory on a crafted XFS image trips the VFS link-count
sanity check:

```
WARNING: fs/inode.c:465 at inc_nlink+0x168/0x180, CPU#2: syz.0.17/25512
RIP: 0010:inc_nlink+0x168/0x180 fs/inode.c:465
Call Trace:
 xfs_bumplink+0x94/0x1e0 fs/xfs/libxfs/xfs_inode_util.c:687
 xfs_dir_create_child+0x340/0x3a0 fs/xfs/libxfs/xfs_dir2.c:872
 xfs_create+0x8bb/0xb80 fs/xfs/xfs_inode.c:728
 xfs_generic_create+0x650/0x800 fs/xfs/xfs_iops.c:216
 xfs_vn_mkdir+0x36/0x50 fs/xfs/xfs_iops.c:308
```

The failing check is the `WARN_ON(!(inode_state_read_once(inode) &
I_LINKABLE))` inside `inc_nlink()` (fs/inode.c:465), reached when
`inode->i_nlink == 0`: `xfs_bumplink()` incremented the link count of an
inode whose link count was already zero and which was not in the
I_LINKABLE (newly created, not yet visible) state. The guest runs with
`panic_on_warn=1`, which escalates this warning to a kernel panic.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y,
CONFIG_XFS_FS=y, booted with `oops=panic` and `panic_on_warn=1`. See
`environment.txt` for the exact QEMU command line and `kernel.config` for
the configuration.

The crash depends on XFS on-disk/in-memory state built up by the preceding
programs, so the reproducer is an ordered program set:

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

`reproducer/repro.kernsight.prog` is the pipeline-verified reproducer (83
state-reconstruction programs followed by the triggering testcase, in
execution order). In the fuzzing pipeline it was confirmed on a fresh VM
(reproducer pipeline outcome: confirmed=true, status=verified). An external
standalone chunked replay on a fresh VM also hit: the warning fired 230.6
seconds after guest boot and `panic_on_warn=1` escalated it to a kernel
panic (full console: `logs/external-verification/
external-verification-serial.log`).

`reproducer/repro.prog` is the minimal triggering testcase extracted by
the pipeline; it depends on the state above and does not reproduce alone.

A C reproducer has not been included yet: syzkaller's reproducer pipeline
is still generating and verifying one; it will be added once it passes.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (83-program ordered set + triggering testcase)
- `reproducer/repro.prog` — minimal triggering testcase (state-dependent; does not reproduce alone)
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics
- `logs/external-verification/external-verification-serial.log` — full serial console of the external verification run (warning at 230.6 s after boot, escalated to panic)

## Notes for maintainers

- What is confirmed: `mkdir()` on a crafted XFS image ends in
  `xfs_bumplink()` -> `inc_nlink()` on an inode with `i_nlink == 0` that
  does not have I_LINKABLE set, tripping the WARN_ON at fs/inode.c:465.
- Natural next step: inspect the XFS directory-create path
  (`xfs_dir_create_child()` -> `xfs_bumplink()`) for how a zero-link inode
  can be used as the parent there; the crafted image contents are in the
  reproducer.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
