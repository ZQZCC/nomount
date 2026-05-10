# NoMount

> WARNING: This project operates directly at the kernel VFS layer and is intended for research and development. Also this project may contain bugs. Proceed with caution.

**NoMount** is a kernel-based file injection and path redirection framework for Android kernels.

Unlike traditional root solutions that rely on `mount --bind` (which pollutes `/proc/mounts`, changes mount namespaces, and is easily detected), NoMount operates **purely at the VFS (Virtual File System) layer**. It manipulates path resolution and directory iteration directly inside the kernel, making injections effective yet virtually invisible to userspace detection methods.

## Why NoMount?

Traditional methods (such a Magic Mount) modify the mount table. Some detectors and banking apps scan `/proc/self/mountinfo` to find these anomalies.

**NoMount changes the paradigm:**

1. **No Mounts:** No `mount()` syscalls are ever used. The mount table remains 100% stock.
2. **Visual Injection:** Uses advanced `iterate_dir` hooking to make "new" files appear in read-only directories (like `/vendor`) without physically touching the partition.
3. **Files Redireccion:** Any file passed through `getname_hook` is intercepted by NoMount, ensuring that any file can be redirected from anywhere.
4. **Native Permission Delegation:** By seamlessly redirecting the underlying inode without permission hooks, it inherently bypasses restrictions while keeping **SELinux** perfectly intact.

## Key Features

* **Transparent Path Redirection:** Intercepts a target VFS path (e.g., `/system/app/YouTube/YouTube.apk`) and redirects the file descriptor to a modified file in a different partition (e.g., `/data`). The userspace process is unaware of the redirection.
* **VFS Directory Injection:** Injects completely new file or directory entries into read-only system paths. Using custom `iterate_dir` kernel hooks, injected files appear natively in standard `readdir` calls, `ls` outputs, and Java `File.list()`.
* **Security Context Bypass:** `inode_permission` and `generic_permission` hooks ensure that the injected files can be traversed and read, while correctly simulating the typical attributes of system partitions.
* **UID-Based Rule Isolation:** Utilizes a hash table to filter active rules per process UID. Specific applications can be isolated to see the 100% stock filesystem without any applied injections.

## Kernel Integration

Please see [kernel/README.md](kernel/README.md) for detailed kernel-side integration, patch instructions, and atchitecture details.

## Usage (Userspace)

The subsystem is controlled via the [`nm`](userspace/src/nm.c) binary communicating through a custom IOCTL interface.

| Command | Syntax | Description |
| :--- | :--- | :--- |
| **Add Rule** | `nm add <virtual> <real>` | Inject `real` file at `virtual` path. |
| **Delete Rule** | `nm del <virtual>` | Remove a specific injection rule. |
| **Remove Rule** | `nm rm <virtual>` | Alias for `del`. |
| **Block UID** | `nm block <uid>` | Isolate UID from seeing any injections. |
| **Unblock UID** | `nm unblock <uid>` | Restore injection visibility for UID. |
| **List Rules** | `nm ls [json]` | Show currently active rules (supports standard or `json` output). |
| **Clear All** | `nm clear` | Flush all rules and UID blocks immediately. |
| **Version** | `nm ver` | Show the kernel subsystem version. |

### Examples

**Inject a custom library:**

```bash
# The system thinks libfoo.so is in /vendor, but it loads from /data
nm add /vendor/lib64/soundfx/libfoo.so /data/local/tmp/my_lib.so

```

**Replace a config file:**

```bash
# Instantly replace audio configs system-wide
nm add /vendor/etc/audio_effects.conf /data/adb/modules/my_mod/audio_effects.conf

```

**Redirect any file:**
```bash
# nm binary recognize relative paths, reconstruct it internally and redirect file correctly
nm add test temp

```

**Hide root from a banking app:**

```bash
# App with UID 10256 will see the stock system, no injections
nm block 10256

```

## Special thanks:

-  **[HymoFS](https://github.com/Anatdx/HymoFS)**: Inspiration for this project.
-  **[A7mdwassa](https://github.com/A7mdwassa)**: Tester and contributor.
-  **[backslashxx](https://github.com/backslashxx)**: Code optimization.
-  **[KernelSU](https://github.com/tiann/KernelSU)**: Root solution.
-  **All testers**: Thanks for making this project more stable!

## Disclaimer

**NoMount** is a powerful kernel modification tool intended for research and development. Modifying kernel behavior carries inherent risks, including system instability or data loss. The developers are not responsible for bricked devices or thermonuclear war.

