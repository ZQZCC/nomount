# NoMount - Detail Hook Architecture & Integration

This document details the hook architecture used by NoMount within the Linux Virtual File System (VFS).


## 1. Initialization and Configuration (`Kconfig` & `Makefile`)
*   **Files:** `fs/Kconfig`, `fs/Makefile`
*   **Purpose:** Register NoMount in kernel.
*   **Mechanism:** Adds the `CONFIG_NOMOUNT` flag to allow enabling/disabling the module during kernel configuration (`menuconfig`) and tells the build system to include `nomount.o` in the final link.
*   **Integration:**:

#### `fs/Kconfig`:

```diff
diff --git a/fs/Kconfig b/fs/Kconfig
index c34b9d4e0..8ec1f2b07 100644
--- a/fs/Kconfig
+++ b/fs/Kconfig
@@ -xxx,xx +xxx,xx @@

+config NOMOUNT
+	bool "NoMount Path Redirection Subsystem"
+	default y
+	help
+	  NoMount allows path redirection and virtual file injection
+	  without mounting filesystems. Useful for systemless modifications.
+
 endmenu
```

#### `fs/Makefile`:

```diff
diff --git a/fs/Makefile b/fs/Makefile
--- a/fs/Makefile
+++ b/fs/Makefile
@@ -xx,x +xx,x @@
 
+obj-$(CONFIG_NOMOUNT) += nomount.o
+
 obj-$(CONFIG_PROC_FS) += proc_namespace.o

```

---

## 2. Path Lookup, Access Control, and Permissions.
*    **Files:** `fs/namei.c`
*    **Hooks:** `getname_flags`, `getname_kernel`, `generic_permission` and `inode_permission`
*    **Purpose:** Intercept text strings (paths) originating from both Userspace and Kernelspace before the VFS converts them into physical structures (`dentry`/`inode`). Additionally, ensure that injected files can be traversed and read while correctly simulating the typical attributes of system partitions.
*   **Mechanism:**
    *   In `namei.c`, `nomount_getname_hook` is executed immediately after the path string is allocated in the kernel. `getname_flags` intercepts standard application traffic, while `getname_kernel` catches internal system requests (e.g., `request_firmware`). If the path matches an active rule, the original string is replaced with the actual path of the redirected file. The rest of the kernel processes the call without knowing that it was tricked.
    *   In `namei.c`, the `nomount_allow_access` hook forces a return of `0` (Success) to prevent native DAC/MAC checks (like SELinux) from blocking access to our injected folders.
*   **Integration:**

#### `fs/namei.c`:

```diff
diff --git a/fs/namei.c b/fs/namei.c
--- a/fs/namei.c
+++ b/fs/namei.c
@@ -xxx,xx +xxx,xx @@
 
 #define EMBEDDED_NAME_MAX	(PATH_MAX - offsetof(struct filename, iname))
 
+#ifdef CONFIG_NOMOUNT
+extern struct filename *nomount_getname_hook(struct filename *name);
+extern int nomount_allow_access(struct inode *inode, int mask);
+#endif
+
 struct filename *
 getname_flags(const char __user *filename, int flags, int *empty)
 {
@@ -xxx,xx +xxx,xx @@ getname_flags(const char __user *filename, int flags, int *empty)
 
 	result->uptr = filename;
 	result->aname = NULL;
+#ifdef CONFIG_NOMOUNT
+	if (!IS_ERR(result)) {
+		result = nomount_getname_hook(result);
+	}
+#endif
 	audit_getname(result);
 	return result;
 }
@@ -xxx,xx +xxx,xx @@ getname_kernel(const char * filename)
 	result->uptr = NULL;
 	result->aname = NULL;
 	result->refcnt = 1;
+#ifdef CONFIG_NOMOUNT
+	if (!IS_ERR(result)) {
+		result = nomount_getname_hook(result);
+	}
+#endif
 	audit_getname(result);
 
 	return result;
@@ -xxx,xx +xxx,xx @@ int generic_permission(struct inode *inode, int mask)
 {
 	int ret;
 
+#ifdef CONFIG_NOMOUNT
+	int nm_perm = nomount_allow_access(inode, mask);
+	if (unlikely(nm_perm < 0)) return nm_perm;
+	if (unlikely(nm_perm > 0)) return 0;
+#endif
+
 	/*
 	 * Do the basic permission checks.
 	 */
@@ -xxx,xx +xxx,xx @@ int inode_permission(struct inode *inode, int mask)
 {
 	int retval;
 
+#ifdef CONFIG_NOMOUNT
+	int nm_perm = nomount_allow_access(inode, mask);
+	if (unlikely(nm_perm < 0)) return nm_perm;
+	if (unlikely(nm_perm > 0)) return 0;
+#endif
+
 	retval = sb_permission(inode->i_sb, inode, mask);
 	if (retval)
 		return retval;
```

---

## 3. Inverse Resolution and Visibility
*   **File:** `fs/d_path.c`
*   **Hook:** `d_path`
*   **Purpose:** Avoid information leaks and falsify real paths through virtual paths.
*   **Mechanism:** If the kernel tries to resolve the path of a file that was injected by us, `nomount_handle_dpath` intercepts the output and returns the "Virtual Path" instead of the actual physical location (e.g. returning `/system/bin/su` instead of `/data/adb/modules/test/su`).
*   **Integration:**

#### `fs/d_path.c`:

```diff
diff --git a/fs/d_path.c b/fs/d_path.c
--- a/fs/d_path.c
+++ b/fs/d_path.c
@@ -xxx,xx +xxx,xx @@ static void get_fs_root_rcu(struct fs_struct *fs, struct path *root)
 	} while (read_seqcount_retry(&fs->seq, seq));
 }
 
+#ifdef CONFIG_NOMOUNT
+extern char *nomount_handle_dpath(const struct path *path, char *buf, int buflen);
+#endif
+
 /**
  * d_path - return the path of a dentry
  * @path: path to report
@@ -xxx,x +xxx,xx @@ char *d_path(const struct path *path, char *buf, int buflen)

+#ifdef CONFIG_NOMOUNT
+	char *nm_path = nomount_handle_dpath(path, buf, buflen);
+	if (unlikely(nm_path)) {
+		return nm_path;
+	}
+#endif
+
 	/*
 	 * We have various synthetic filesystems that never get mounted.  On
 	 * these filesystems dentries are never used for lookup purposes, and
```

---

## 4. Directory Listing
*   **File:** `fs/readdir.c`
*   **Hooks:** `iterate_dir` (via the `nomount_handle_iterate_dir` macro)
*   **Purpose:** Allow commands like `ls` or listing APIs to seamlessly see virtual files injected into legitimate system directories.
*   **Mechanism:** Instead of manually manipulating userspace buffers in the `getdents` syscalls, NoMount now hooks directly into the directory iteration engine (`iterate_dir`). The wrapper macro acts as a smart proxy: it first allows the native filesystem to read the physical disk without interference. By tracking the directory offset (`ctx->pos`), NoMount detects when the native iteration reaches the End of File (EOF) or pauses. Once the physical files are fully processed, it calls `nomount_vfs_inject_dir` to seamlessly inject virtual entries using the kernel's native `dir_emit()` function.
*   **Integration:**

#### `fs/readdir.c`:

```diff
diff --git a/fs/readdir.c b/fs/readdir.c
--- a/fs/readdir.c
+++ b/fs/readdir.c
@@ -xx,xx +xx,xx @@
 	unsafe_copy_to_user(dst, src, len, label);		\
 } while (0)
 
+#ifdef CONFIG_NOMOUNT
+extern void nomount_vfs_inject_dir(struct file *file, struct dir_context *ctx);
+extern bool nomount_should_skip(void);
+extern const loff_t nomount_magic_pos;
+
+#define nomount_handle_iterate_dir(file, ctx, shared, res)               \
+do {                                                                     \
+    loff_t _old_pos = (ctx)->pos;                                        \
+    bool _nm_skip = nomount_should_skip();                               \
+                                                                         \
+    if ((ctx)->pos >= nomount_magic_pos && !_nm_skip) {                  \
+        (res) = 0;                                                       \
+    } else {                                                             \
+        if (shared)                                                      \
+            (res) = (file)->f_op->iterate_shared((file), (ctx));         \
+        else                                                             \
+            (res) = (file)->f_op->iterate((file), (ctx));                \
+    }                                                                    \
+                                                                         \
+    if ((res) >= 0 && !_nm_skip) {                                       \
+        if ((ctx)->pos == _old_pos || (ctx)->pos >= nomount_magic_pos) { \
+            nomount_vfs_inject_dir((file), (ctx));                       \
+        }                                                                \
+    }                                                                    \
+} while (0)
+#endif
 
 int iterate_dir(struct file *file, struct dir_context *ctx)
 {
@@ -xx,xx +xx,xx @@ int iterate_dir(struct file *file, struct dir_context *ctx)
 	res = -ENOENT;
 	if (!IS_DEADDIR(inode)) {
 		ctx->pos = file->f_pos;
+#ifdef CONFIG_NOMOUNT
+		nomount_handle_iterate_dir(file, ctx, shared, res);
+#else
 		if (shared)
 			res = file->f_op->iterate_shared(file, ctx);
 		else
 			res = file->f_op->iterate(file, ctx);
+#endif
 		file->f_pos = ctx->pos;
 		fsnotify_access(file);
 		file_accessed(file);
```

---

## 5. Metadata Spoofing (Stat & Mmap)
To be undetectable, the metadata of the files and file systems must match their virtual location.

*   **File Attributes (`fs/stat.c`):**
    *   **Hook:** `vfs_getattr`
    *   **Mechanism:** A *Wrapper* pattern is used. The `nomount_handle_getattr` hook captures the native output of `vfs_getattr_nosec`. If the original read was successful, it overwrites the `inode` and `device id` (dev) in the `kstat` structure before returning it to the user.
	*   **Integration:**

#### `fs/stat.c`

```diff
diff --git a/fs/stat.c b/fs/stat.c
--- a/fs/stat.c
+++ b/fs/stat.c
@@ -xxx,xx +xxx,xx @@ int vfs_getattr_nosec(const struct path *path, struct kstat *stat,
 }
 EXPORT_SYMBOL(vfs_getattr_nosec);
 
+#ifdef CONFIG_NOMOUNT
+extern int nomount_handle_getattr(int ret, const struct path *path, struct kstat *stat);
+#endif
+
 /*
  * vfs_getattr - Get the enhanced basic attributes of a file
  * @path: The file of interest
@@ -xxx,xx +xxx,xx @@ int vfs_getattr(const struct path *path, struct kstat *stat,
 	retval = security_inode_getattr(path);
 	if (retval)
 		return retval;
+#ifdef CONFIG_NOMOUNT
+        return nomount_handle_getattr(vfs_getattr_nosec(path, stat, request_mask, query_flags), path, stat);
+#else
 	return vfs_getattr_nosec(path, stat, request_mask, query_flags);
+#endif
 }
 EXPORT_SYMBOL(vfs_getattr);
 
```

*   **Memory Maps (`fs/proc/task_mmu.c`):**
    *   **Hook:** `show_map_vma`
    *   **Mechanism:** When a process reads `/proc/self/maps`, the kernel prints the loaded libraries into memory. The `nomount_spoof_mmap_metadata` hook replaces the raw device and inode just before they are printed to the `seq_file`.
	*   **Integration:**

#### `fs/proc/task_mmu.c`:

```diff
diff --git a/fs/proc/task_mmu.c b/fs/proc/task_mmu.c
--- a/fs/proc/task_mmu.c
+++ b/fs/proc/task_mmu.c
@@ -xxx,xx +xxx,xx @@ static void show_vma_header_prefix(struct seq_file *m,
 	seq_putc(m, ' ');
 }
 
+#ifdef CONFIG_NOMOUNT
+extern bool nomount_spoof_mmap_metadata(struct inode *inode, dev_t *dev, unsigned long *ino);
+#endif
+
 static void
 show_map_vma(struct seq_file *m, struct vm_area_struct *vma)
 {
@@ -xxx,xx +xxx,xx @@ show_map_vma(struct seq_file *m, struct vm_area_struct *vma)
 		struct inode *inode = file_inode(vma->vm_file);
 		dev = inode->i_sb->s_dev;
 		ino = inode->i_ino;
+#ifdef CONFIG_NOMOUNT
+		nomount_spoof_mmap_metadata(inode, &dev, &ino);
+#endif
 		pgoff = ((loff_t)vma->vm_pgoff) << PAGE_SHIFT;
 	}

```

*   **Filesystem Attributes (`fs/statfs.c`):**
    *   **Hook:** `vfs_statfs`
    *   **Mechanism:** Alter the `kstatfs` structure. If an application requests partition metadata (e.g. to check if the file is on an `ext4` or `erofs` volume), the `nomount_spoof_statfs` hook injects the "Magic Number" (FS type) corresponding to the desired virtual partition.
	*  **Integration:**

#### `fs/statfs.c`:

```diff
diff --git a/fs/statfs.c b/fs/statfs.c
--- a/fs/statfs.c
+++ b/fs/statfs.c
@@ -xx,xx +xx,xx @@
 #include <linux/compat.h>
 #include "internal.h"
 
+#ifdef CONFIG_NOMOUNT
+extern void nomount_spoof_statfs(const struct path *path, struct kstatfs *buf);
+#endif
+
 static int flags_by_mnt(int mnt_flags)
 {
 	int flags = 0;
@@ -xxx,xx +xxx,xx @@ int vfs_statfs(const struct path *path, struct kstatfs *buf)
 	error = statfs_by_dentry(path->dentry, buf);
 	if (!error)
 		buf->f_flags = calculate_f_flags(path->mnt);
+#ifdef CONFIG_NOMOUNT
+	nomount_spoof_statfs(path, buf);
+#endif
 	return error;
 }
 EXPORT_SYMBOL(vfs_statfs);

```
---
