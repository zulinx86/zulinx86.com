---
layout: post
title: sysfs
---



## What is sysfs?
- sysfs is a RAM-based virtual filesystem.
- It provides a means to export kernel data structure, their attributes, and the linkages between them to userspace, including the device model.
- It is tied inherently to the kobject infrastructure. Please see the [kobject](https://zulinx86.com/notebook/kernel/kobject) page for details.
- It is tied inherently to the kernfs infrastructure too. Please see the [kernfs](https://zulinx86.com/notebook/kernel/kernfs) page for details.



## Usage
### Preparation
1. Compile the kernel with `CONFIG_SYSFS`.
```
Symbol: SYSFS [=y]
Type  : bool
Defined at fs/sysfs/Kconfig:2
  Prompt: sysfs file system support
  Visible if : EXPERT [=n]
  Location:
    Main menu
      -> File systems
(1)     -> Pseudo filesystems
```
1. Mount sysfs (if not mounted).
```
# mount -t sysfs sysfs /sys
# mount | grep sysfs
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
```



## Basic Structure
- Directory
	- kobjects (`struct kobject`) are exported as directories in sysfs.
	- The directory is created as a subdirectory of the kobject's parent.
- File
	- attributes (`struct attribute`) are exported as regular files in sysfs.
	- I/O operations to those attributes are defined by `struct sysfs_ops`.


### Top Level Directory Layout
```
$ ls -l /sys/
total 0
drwxr-xr-x  2 root root 0 May 17 19:22 block
drwxr-xr-x 24 root root 0 May 17 19:22 bus
drwxr-xr-x 30 root root 0 May 17 19:22 class
drwxr-xr-x  4 root root 0 May 17 19:22 dev
drwxr-xr-x 14 root root 0 May 17 19:22 devices
drwxr-xr-x  5 root root 0 May 17 19:22 firmware
drwxr-xr-x  6 root root 0 May 17 19:22 fs
drwxr-xr-x  2 root root 0 May 17 19:22 hypervisor
drwxr-xr-x 14 root root 0 May 17 19:22 kernel
drwxr-xr-x 87 root root 0 May 17 19:22 module
drwxr-xr-x  3 root root 0 May 18 20:20 power
```

- `devices/`
	- It maps to directly to the internal kernel device tree, which is a hierarchy of `struct device`.
- `bus/`
	- It contains various bus types in the kernel.
	- Each bus's directory contains two subdirectories:
		- `devices/`
			- It contains symlinks for each device discovered in the system.
		- `drivers/`
			- It contains a directory for each device driver that is loaded for devices on that particular bus.
- `dev/`
	- It contains two directories `char/` and `block/`.
	- Inside these two directories, there are symlinks named `<major>:<minor>`.
	- These symlinks point to the sysfs directory for the given device.



## Data Structures
### Filesystem Type
[https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/mount.c#L90](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/mount.c#L90)
```c
static struct file_system_type sysfs_fs_type = {
	.name			= "sysfs",
	.init_fs_context	= sysfs_init_fs_context,
	.kill_sb		= sysfs_kill_sb,
	.fs_flags		= FS_USERNS_MOUNT,
};
```

[https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/mount.c#L55](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/mount.c#L55)
```c
static int sysfs_init_fs_context(struct fs_context *fc)
{
	struct kernfs_fs_context *kfc;
	struct net *netns;

	if (!(fc->sb_flags & SB_KERNMOUNT)) {
		if (!kobj_ns_current_may_mount(KOBJ_NS_TYPE_NET))
			return -EPERM;
	}

	kfc = kzalloc(sizeof(struct kernfs_fs_context), GFP_KERNEL);
	if (!kfc)
		return -ENOMEM;

	kfc->ns_tag = netns = kobj_ns_grab_current(KOBJ_NS_TYPE_NET);
	kfc->root = sysfs_root;
	kfc->magic = SYSFS_MAGIC;
	fc->fs_private = kfc;
	fc->ops = &sysfs_fs_context_ops;
	if (netns) {
		put_user_ns(fc->user_ns);
		fc->user_ns = get_user_ns(netns->user_ns);
	}
	fc->global = true;
	return 0;
}

static void sysfs_kill_sb(struct super_block *sb)
{
	void *ns = (void *)kernfs_super_ns(sb);

	kernfs_kill_sb(sb);
	kobj_ns_drop(KOBJ_NS_TYPE_NET, ns);
}
```


### `struct attribute`
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L30](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L30)
```c
struct attribute {
	const char		*name;
	umode_t			mode;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	bool			ignore_lockdep:1;
	struct lock_class_key	*key;
	struct lock_class_key	skey;
#endif
};
```

#### `struct bus_attribute`
`struct bus_attribute` for `struct bus`  
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/bus.h#L123](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/bus.h#L123)
```c
struct bus_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct bus_type *bus, char *buf);
	ssize_t (*store)(struct bus_type *bus, const char *buf, size_t count);
};
```

#### `struct class_attribute`
`struct class_attribute` for `struct class`  
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/class.h#L191](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/class.h#L191)
```c
struct class_attribute {
	struct attribute attr;
	ssize_t (*show)(struct class *class, struct class_attribute *attr,
			char *buf);
	ssize_t (*store)(struct class *class, struct class_attribute *attr,
			const char *buf, size_t count);
};
```

#### `struct device_attribute`
`struct device_attribute` for `struct device`  
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device.h#L100](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device.h#L100)
```c
/* interface for exporting device attributes */
struct device_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct device *dev, struct device_attribute *attr,
			char *buf);
	ssize_t (*store)(struct device *dev, struct device_attribute *attr,
			 const char *buf, size_t count);
};
```

#### `struct driver_attribute`
`struct driver_attribute` for `struct device_driver`  
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/driver.h#L135](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/driver.h#L135)
```c
struct driver_attribute {
	struct attribute attr;
	ssize_t (*show)(struct device_driver *driver, char *buf);
	ssize_t (*store)(struct device_driver *driver, const char *buf,
			 size_t count);
};
```


### `struct bin_attribute`
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L175](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L175)
```c
struct bin_attribute {
	struct attribute	attr;
	size_t			size;
	void			*private;
	struct address_space *(*f_mapping)(void);
	ssize_t (*read)(struct file *, struct kobject *, struct bin_attribute *,
			char *, loff_t, size_t);
	ssize_t (*write)(struct file *, struct kobject *, struct bin_attribute *,
			 char *, loff_t, size_t);
	int (*mmap)(struct file *, struct kobject *, struct bin_attribute *attr,
		    struct vm_area_struct *vma);
};
```


### `struct attribute_group`
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L84](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L84)
```c
/**
 * struct attribute_group - data structure used to declare an attribute group.
 * @name:	Optional: Attribute group name
 *		If specified, the attribute group will be created in
 *		a new subdirectory with this name.
 * @is_visible:	Optional: Function to return permissions associated with an
 *		attribute of the group. Will be called repeatedly for each
 *		non-binary attribute in the group. Only read/write
 *		permissions as well as SYSFS_PREALLOC are accepted. Must
 *		return 0 if an attribute is not visible. The returned value
 *		will replace static permissions defined in struct attribute.
 * @is_bin_visible:
 *		Optional: Function to return permissions associated with a
 *		binary attribute of the group. Will be called repeatedly
 *		for each binary attribute in the group. Only read/write
 *		permissions as well as SYSFS_PREALLOC are accepted. Must
 *		return 0 if a binary attribute is not visible. The returned
 *		value will replace static permissions defined in
 *		struct bin_attribute.
 * @attrs:	Pointer to NULL terminated list of attributes.
 * @bin_attrs:	Pointer to NULL terminated list of binary attributes.
 *		Either attrs or bin_attrs or both must be provided.
 */
struct attribute_group {
	const char		*name;
	umode_t			(*is_visible)(struct kobject *,
					      struct attribute *, int);
	umode_t			(*is_bin_visible)(struct kobject *,
						  struct bin_attribute *, int);
	struct attribute	**attrs;
	struct bin_attribute	**bin_attrs;
};
```


### `struct sysfs_ops`
https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L238
```c
struct sysfs_ops {
	ssize_t	(*show)(struct kobject *, struct attribute *, char *);
	ssize_t	(*store)(struct kobject *, struct attribute *, const char *, size_t);
};
```



## Methods
- [`void sysfs_init(void)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/mount.c#L97): Initialize sysfs.
	- Create the root of sysfs by `kernfs_create_root()` and set global variable `struct kernfs_root *sysfs_root;`.
	- Set global variable `struct kernfs_node *sysfs_root_kn = sysfs_root->kn;`.
	- Register a new filesystem with `&sysfs_fs_type` by `register_filesystem()`.
- [`int sysfs_create_dir_ns(struct kobject *kobj, const void *ns)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/dir.c#L40): Create a sysfs directory for the `kobj` with namespace.
	- If `kobj->parent` is set, use `kobj->parent->sd` as parent's `struct kernfs_node`.
	- If `kobj->parent` is not set, use `sysfs_root_kn` as parent's `struct kernfs_node`.
	- Create a kernfs directory for this sysfs directory by `kernfs_create_dir_ns()`.
	- Save the pointer to the created `kernfs_node` into `kobj->sd`.
- [`int sysfs_rename_dir_ns(struct kobject *kobj, const char *new_name, const void *new_ns);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/dir.c#L105): Rename a sysfs directory.
	- Save the current kernfs parent of the kernfs directory `kobj->sd`.
	- Rename the kernfs directory `kobj->sd` by `kernfs_rename_ns()`.
- [`int sysfs_move_dir_ns(struct kobject *kobj, struct kobject *new_parent_kobj, const void *new_ns);`]: Move a sysfs directory.
	- Move the kernfs directory `kobj->sd` by `kernfs_rename_ns()`.
- [`int sysfs_create_files(strcut kobject *kobj, const struct attribute * const *ptr)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/file.c#L359): Create sysfs files under the `kobj`.
	- Call `sysfs_create_file()` for each attribute.
- [`int sysfs_create_file(struct kobject *kobj, const struct attribute *attr)`](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L604): Create a sysfs file under the `kobj`.
	- Call `sysfs_create_file_ns()` without namespace.
- [`int sysfs_create_file_ns(struct kobject *kobj, const struct attribute *attr, const void *ns)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/file.c#L345): Create a sysfs file under the `kobj` with namespace.
	- If `kobj->parent` is not set, use sysfs root as parent.
	- Get UID and GID for the `kobj` by `kobject_get_ownership()`.
	- Add file into sysfs by `sysfs_add_file_mode_ns()`. Set `kobj->sd` as the return value.
- [`int sysfs_add_file_mode_ns(struct kernfs_node *parent, const struct attribute *attr, umode_t mode, kuid_t uid, kgid_t gid, const void *ns)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/file.c#L254): Create a sysfs file under the `parent` with mode and namespace.
	- Initialize local variables `kobj`, `sysfs_ops` and `ops`.
	- Create a kernfs file by `__kernfs_create_file()`.
- [`void sysfs_remove_files(struct kobject *kobj, const struct attribute * const *ptr);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/file.c#L519)
	- Call `sysfs_remove_file()` for each attribute.
- [`void sysfs_remove_file(struct kobject *kobj, const struct attribute *attr);`](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L610)
	- Call `sysfs_remove_file_ns()` without namespace.
- [`void sysfs_remove_file_ns(struct kobject *kobj, const struct attribute *attr, const void *ns);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/file.c#L486)
	- Remove a kernfs file by `kernfs_remove_by_name_ns(parent, attr->name, ns)`.
- [`int sysfs_create_bin_file(struct kobject *kobj, const struct bin_attribute *attr);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/file.c#L558)
	- Call `sysfs_add_bin_file_mode_ns()`.
- [`int sysfs_add_bin_file_mode_ns(struct kernfs_node *parent, const struct bin_attribute *battr, umode_t mode, kuid_t uid, kgid_t gid, const void *ns);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/file.c#L304)
	- Create a kernfs file by `__kernfs_create_file()`.
- [`void sysfs_remove_bin_file(struct kobject *kobj, const struct bin_attribute *attr);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/file.c#L578)
	- Call `kernfs_remove_by_name()`.
- [`int sysfs_create_link(struct kobject *kobj, struct kobject *target, const char *name);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/symlink.c#L89)
	- Call `sysfs_do_create_link()` with warning.
- [`int sysfs_create_link_nowarn(struct kobject *kobj, struct kobject *target, const char *name);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/symlink.c#L105)
	- Call `sysfs_do_create_link()` without warning.
- [`int sysfs_do_create_link(struct kobject *kobj, struct kobject *target, const char *name, int warn);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/symlink.c#L67)
	- Call `sysfs_do_create_link_sd()`.
- [`int sysfs_do_create_link_sd(struct kernfs_node *parent, struct kobject *target_kobj, const char *name, int warn);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/symlink.c#L20)
	- Increment the refcount of the target kernfs node.
	- Create a kernfs link by `kernfs_create_link()`.
- [`void sysfs_remove_link(struct kobject *kobj, const char *name);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/sysfs/symlink.c#L143)
	- Call `kernfs_remove_by_name()`.



## Links
- [sysfs.rst - Documentation/filesystems/sysfs.rst - Linux source code (v5.17.9) - Bootlin](https://elixir.bootlin.com/linux/latest/source/Documentation/filesystems/sysfs.rst)
- [kobjects and sysfs - LWN.net](https://lwn.net/Articles/54651/)