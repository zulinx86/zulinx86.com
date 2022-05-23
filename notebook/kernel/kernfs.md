---
layout: post
title: kernfs
---


## What is kernfs?
- kernfs is a set of functions that contain the functionality required for creating the pseudo file systems used internally by various kernel subsystem so that they may utilize virtual files.
- The creation of kernfs resulted from splitting off part of sysfs.



## Data Structures
### `struct kernfs_node`
- `struct kernfs_node` is the building block of kernfs hierarchy.

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L131](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L131)
```c
/*
 * kernfs_node - the building block of kernfs hierarchy.  Each and every
 * kernfs node is represented by single kernfs_node.  Most fields are
 * private to kernfs and shouldn't be accessed directly by kernfs users.
 *
 * As long as count reference is held, the kernfs_node itself is
 * accessible.  Dereferencing elem or any other outer entity requires
 * active reference.
 */
struct kernfs_node {
	atomic_t		count;
	atomic_t		active;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map	dep_map;
#endif
	/*
	 * Use kernfs_get_parent() and kernfs_name/path() instead of
	 * accessing the following two fields directly.  If the node is
	 * never moved to a different parent, it is safe to access the
	 * parent directly.
	 */
	struct kernfs_node	*parent;
	const char		*name;

	struct rb_node		rb;

	const void		*ns;	/* namespace tag */
	unsigned int		hash;	/* ns + name hash */
	union {
		struct kernfs_elem_dir		dir;
		struct kernfs_elem_symlink	symlink;
		struct kernfs_elem_attr		attr;
	};

	void			*priv;

	/*
	 * 64bit unique ID.  On 64bit ino setups, id is the ino.  On 32bit,
	 * the low 32bits are ino and upper generation.
	 */
	u64			id;

	unsigned short		flags;
	umode_t			mode;
	struct kernfs_iattrs	*iattr;
};
```

- `const char *name;`: Name.
- `struct kernfs_node *parent;`: Pointer to its parent.
- `struct kernfs_elem_*`: Type-specific data
	- `struct kernfs_elem_dir dir;`: for `KERNFS_DIR`.
	- `struct kernfs_elem_symlink symlink;`: for `KERNFS_LINK`.
	- `struct kernfs_elem_attr attr;`: for `KERNFS_FILE`.


### `enum kernfs_node_type`
- Types of `kernfs_node`.
	- Directory (`KERNFS_DIR`)
	- File (`KERNFS_FILE`)
	- Symlink (`KERNFS_LINK`)

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L37](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L37)
```c
enum kernfs_node_type {
	KERNFS_DIR		= 0x0001,
	KERNFS_FILE		= 0x0002,
	KERNFS_LINK		= 0x0004,
};
```

#### `struct kernfs_elem_dir`
- Type-specific data structure for `KERNFS_DIR`.

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L94](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L94)
```c
struct kernfs_elem_dir {
	unsigned long		subdirs;
	/* children rbtree starts here and goes through kn->rb */
	struct rb_root		children;

	/*
	 * The kernfs hierarchy this directory belongs to.  This fits
	 * better directly in kernfs_node but is here to save space.
	 */
	struct kernfs_root	*root;
	/*
	 * Monotonic revision counter, used to identify if a directory
	 * node has changed during negative dentry revalidation.
	 */
	unsigned long		rev;
};
```

#### `struct kernfs_elem_attr`
- Type-specific data structure for `KERNFS_FILE`.

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L115](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L115)
```c
struct kernfs_elem_attr {
	const struct kernfs_ops	*ops;
	struct kernfs_open_node	*open;
	loff_t			size;
	struct kernfs_node	*notify_next;	/* for kernfs_notify() */
};
```

#### `struct kernfs_elem_symlink`
- Type-specific data structure for `KERNFS_LINK`.

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L111](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L111)
```c
struct kernfs_elem_symlink {
	struct kernfs_node	*target_kn;
};
```


### `struct kernfs_root`
- `struct kernfs_root` is the root directory of the kernfs.

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L188](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L188)
```c
struct kernfs_root {
	/* published fields */
	struct kernfs_node	*kn;
	unsigned int		flags;	/* KERNFS_ROOT_* flags */

	/* private fields, do not use outside kernfs proper */
	struct idr		ino_idr;
	u32			last_id_lowbits;
	u32			id_highbits;
	struct kernfs_syscall_ops *syscall_ops;

	/* list of kernfs_super_info of this root, protected by kernfs_rwsem */
	struct list_head	supers;

	wait_queue_head_t	deactivate_waitq;
	struct rw_semaphore	kernfs_rwsem;
};
```
- `struct kernfs_node *kn;`: Pointer to its `kernfs_node`.
- `unsigned int flags;`: Flags.
- `struct idr ino_idr;`: ID allocater for this kernfs.
- `struct rw_semaphore kernfs_rwsem;`: Read-write semaphore of this kernfs.


### `struct kernfs_super_info`
[https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/kernfs-internal.h#L56](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/kernfs-internal.h#L56)
```c
struct kernfs_super_info {
	struct super_block	*sb;

	/*
	 * The root associated with this super_block.  Each super_block is
	 * identified by the root and ns it's associated with.
	 */
	struct kernfs_root	*root;

	/*
	 * Each sb is associated with one namespace tag, currently the
	 * network namespace of the task which mounted this kernfs
	 * instance.  If multiple tags become necessary, make the following
	 * an array and compare kernfs_node tag against every entry.
	 */
	const void		*ns;

	/* anchored at kernfs_root->supers, protected by kernfs_rwsem */
	struct list_head	node;
};
```


### `struct kernfs_syscall_ops`
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L176](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kernfs.h#L176)
```c
/*
 * kernfs_syscall_ops may be specified on kernfs_create_root() to support
 * syscalls.  These optional callbacks are invoked on the matching syscalls
 * and can perform any kernfs operations which don't necessarily have to be
 * the exact operation requested.  An active reference is held for each
 * kernfs_node parameter.
 */
struct kernfs_syscall_ops {
	int (*show_options)(struct seq_file *sf, struct kernfs_root *root);

	int (*mkdir)(struct kernfs_node *parent, const char *name,
		     umode_t mode);
	int (*rmdir)(struct kernfs_node *kn);
	int (*rename)(struct kernfs_node *kn, struct kernfs_node *new_parent,
		      const char *new_name);
	int (*show_path)(struct seq_file *sf, struct kernfs_node *kn,
			 struct kernfs_root *root);
};
```



## Methods
- High-Level
	- [`struct kernfs_root *kernfs_create_root(struct kernfs_syscall_ops *scops, unsigend int flags, void *priv);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L907)
		- Allocate memory for `struct kernfs_root root;`.
		- Initialize some data structures for `root`.
			- Initialize IDR (`root->ino_idr`).
			- Initialize read-write semaphore (`root->kernfs_rwsem`).
			- Initialize list of `kernfs_super_info` of this root (`root->supers`).
		- Create a new directory-type kernfs node by `__kernfs_new_node()`.
	- [`struct kernfs_node *kernfs_new_node(struct kernfs_node *parent, const char *name, umode_t mode, kuid_t uid, kgid_t gid, unsigned flags);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L640)
		- Create a new kernfs node by `__kernfs_new_node()`.
		- Increment the refcount of `parent`.
		- Set the parent of the new kernfs node as `parent`.
- Low-Level
	- [`struct kernfs_node *__kernfs_new_node(struct kernfs_root *root, struct kernfs_node *parent, const char *name, umode_t mode, kuid_t uid, kgid_t gid, unsigned flags);`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L571)
		- 

- Methods
	- [`struct kernfs_root *kernfs_create_root(struct kernfs_syscall_ops *sops, unsigned int flags, void *priv)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L907): Create a new kernfs hierarchy.
	- [`void kernfs_destroy_root(struct kernfs_root *root)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L962): Destroy a kernfs hierarchy.
	- [`struct kernfs_root *kernfs_root(struct kernfs_node *kn)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/kernfs-internal.h#L45): Get the `kernfs_root` which the `kernfs_node` belongs to.


- Methods
	- High-Level
		- [`struct kernfs_node *kernfs_create_dir_ns(struct kernfs_node *parent, const char *name, umode_t mode, kuid_t uid, kgid_t gid, void *priv, const void *ns)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L985): Create a new directory.
			- Create a new node for the directory by `kernfs_new_node()`.
			- Set members of the new node, including directory-type specific ones.
			- Link the new node into the children list of its parent by `kernfs_add_one()`.
		- [`struct kernfs_node *kernfs_create_empty_dir(struct kernfs_node *parent, const char *name)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L962): Create an always empty directory.
			- Implementation is very similar to `kernfs_create_dir_ns()` except for that it's an empty directory.
		- [`struct kernfs_node *__kernfs_create_file(struct kernfs_node *parent, const char *name, umode_t mode, kuid_t uid, kgid_t gid, loff_t size, const struct kernfs_ops *ops, void *priv, const void *ns, struct lock_class_key *key)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/file.c#L973): Craete a new file.
			- Create a new node for the file by `kernfs_new_node()`.
			- Set members of the new node, including file-type specific ones.
			- Link the new node into the chilren list of its parent by `kernfs_add_one()`.
		- [`struct kernfs_node *kernfs_create_link(struct kernfs_node *parent, const char *name, struct kernfs_node *target)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/symlink.c#L25)
			- Create a new node for the link by `kernfs_new_node()`.
			- Set members of the new node, including link-type specific ones.
			- Increment the refcount of the target `kernfs_node`.
			- Link the new node into the children list of its parent by `kernfs_add_one()`.
	- Low-Level
		- [`struct kernfs_add_one(struct kernfs_node *kn)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L718): Link `kernfs_node` into the children list of its parent.
			- Acquire write-lock of read-write semaphore.
			- Set hash (`kn->hash`) calculated by `kernfs_name_hash()`.
			- Insert the `kernfs_node` into sibling rbtree by `kernfs_link_sibling()`.
			- Update timestamps of its parent.
			- Release write-lock of read-write semaphore.
		- [`int kernfs_link_sibling(struct kernfs_node *kn)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L347): Insert `kernfs_node` into sibling rbtree (children rbtree of its parent).
		- [`void kernfs_get(struct kernfs_node *kn)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L493): Increment the refcount of the given `kernfs_node`.
		- [`void kernfs_put(struct kernfs_node *kn)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L508): Decrement the refcount of the given `kernfs_node`.
		- [`unsigned int kernfs_name_hash(const char *name, const void *ns)`](https://elixir.bootlin.com/linux/v5.17.8/source/fs/kernfs/dir.c#L298): Return 31-bit hash of ns + name.

