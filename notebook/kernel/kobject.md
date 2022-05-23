---
layout: post
title: kobject, kset, kobj_type, kref
---



## General
- The linux kernel driver model is built upon the kobject abstraction.
- The kobject abstraction provides a hierarchical structure embeding reference counter.



## `struct kobject`
- It includes a reference counter `kref` and some links to make hierarchy.
- It is not used alone, but embedded within other data structure.

https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L64
```c
struct kobject {
	const char		*name;
	struct list_head	entry;
	struct kobject		*parent;
	struct kset		*kset;
	const struct kobj_type	*ktype;
	struct kernfs_node	*sd; /* sysfs directory entry */
	struct kref		kref;
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
	struct delayed_work	release;
#endif
	unsigned int state_initialized:1;
	unsigned int state_in_sysfs:1;
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};
```

- Members
	- `const char *name;`: Name of this `kobject`.
	- `const struct kobj_type *ktype;`: Type of `kobject`.
	- `struct kref kref;`: Reference counter.
	- `struct kobject *parent;`: Pointer to its parent (allowing it to be arranged into hierarchy).
	- `struct kset *kset;`: Pointer to a `kset` including this `kobject`.
	- `struct list_head entry;`: Node entry of `kset`'s list including this `kobject`.
	- `struct kernfs_node *sd;`: Pointer to its kernfs node (sysfs directory entry).
- Methods
	- High-Level
		- [`struct kobject *kobject_create(void)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L780): Create a new `kobject` dynamically.
			- Allocate memory for a new `kobject` by `kzalloc()`.
			- Initialize the new `kobject` with `dynamic_kobj_ktype` by `kobject_init()`.
		- [`void kobject_init(struct kobject *kobj, const struct kobj_type *ktype)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L349): Initialize the `kobj`.
			- Initialize `kobj`'s members by `kobject_init_internal()`.
			- Set `kobj->ktype` as the given value.
		- [`int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L426): Add the `kobj` into the hierarchy.
			- Call `kobject_add_vargs()`.
		- [`struct kobject *kobject_create_and_add(const char *name, struct kobject *parent)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L805): `kobject_create()` + `kobject_add()`.
		- [`int kobject_init_and_add(struct kobject *kobj, const struct kobj_type *ktype, struct kobject *parent, const char *fmt, ...)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L464): `kobject_init()` + `kobject_add()`.
		- [`int kobject_uevent(struct kobject *kobj, enum kobject_action action)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject_uevent.c#L640): Notify userspace by sending an uevent.
			- Call `kobject_uevent_env()`.
	- Low-Level
		- [`void kobject_init_internal(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L212)
			- Initialize a refcount (`kobj->kerf`).
			- Initialize a list head (`kobj->entry`).
			- Initialize its states (`kobj->state_*`).
		- [`int kobject_add_varg(struct kobject *kobj, struct kobject *parent, const char *fmt, va_list vargs)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L378)
			- Set the name of the `kobj` by `kobject_set_name_vargs()`.
			- Set `kobj->parent` as the given `parent`.
			- Add the `kobj` into the hierarchy by `kobject_add_internal()`.
		- [`int kobject_set_name_vargs(struct kobject *kobj, const char *fmt, va_list vargs)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L281): Set the name of the `kobj`.
		- [`int kobject_add_internal(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L225): Add itself into hierarchy.
			- If `kobj->parent` is set
				- Increment the refcount of `kobj->parent` by `kobject_get()`.
			- If `kobj->kset` is set,
				- If `kobj->parent` is not set, set `kobj->parent` as `&kobj->kset->kobj`.
				- Join the `kobj` into the list of `kobj->kset` by `kobj_kset_join()`.
		- [`void kobj_kset_join(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L189): Join the `kobj` into the list of `kobj->kset`.
			- Increment the refcount of `kobj->kset`.
			- Add `kobj` to the tail of `kobj->kset->list`.
			- Create sysfs directory by `create_dir()`.
		- [`void kobj_kset_leave(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L201): Remove the `kobj` from the list of `kobj->kset`.
			- Delete the `kobj` from the `kobj->kset`.
			- Decrement the refcount of the `kobj->kset`.
		- [`int create_dir(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L83): Create sysfs directory.
			- Create a new sysfs directory by `sysfs_create_dir_ns()`.
			- Create default attributes (sysfs files) under the new sysfs directory by `populate_dir()`.
			- Create default attribute groups under the new sysfs directory by `sysfs_create_groups()`.
			- Increment the refcount of `kobj->sd` so that it stays even when its ancestor goes away by `sysfs_get()`.
			- If `kobj` has namespace operations, enable namespace support by `sysfs_enable_ns()`.
		- [`int populate_dir(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L66): Create default attributes (sysfs files) under the `kobj` (sysfs directory).
			- Iterate all attributes (`kobj->ktype->default_attrs`).
				- Create a sysfs file for each attribute by `sysfs_create_file()`.
		- [`struct kobject *kobject_get(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L652): Increment the refcount of the `kobj`.
			- Increment the refcount of the `kobj`.
		- [`void kobject_put(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L746): Decrement the refcount of the `kobj`.
			- Decrement the refcount of the `kobj` by `kref_put()`.
				- When the refcount equals to 0, it invokes `kobject_release()`.
		- [`void kobject_release(struct kref *kref)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L725): Release the `kobj`.
			- Call `kobject_cleanup()`.
		- [`void kobject_cleanup(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L679)
			- If `kobj->state_in_sysfs` is true, unlink the `kobj` from hierarchy by `__kobject_del()`.
			- It `kobj->ktype` is set and `kobj->ktype->release` is set, call `kobj->ktype->release()`.
			- Decrement the refcount of its parent.
		- [`void kobject_del(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L635): Unlink the `kobj` from hierarchy.
			- Unlink the `kobj` from hierarchy by `__kobject_del()`.
			- Decrement the refcount of its parent.
		- [`void __kobject_del(struct kobject *kobj)`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L602): Unlink the `kobj` from hierarchy.
			- If `kobj->ktype` is set, remove default attribute groups from sysfs by `sysfs_remove_groups()`.
			- If `kobj->state_add_uevent_sent` is true and `kobj->state_remove_uevent_sent` is false, send an remove uevent by `kobject_uevent()`.
			- Clean up sysfs.
				- Remove the sys directory corresponding the `kobj` by `sysfs_remove_dir()`.
				-  Decrement the refcount of `kobj->sd` by `sysfs_put()`.
			- Clean up itself.
				- `kobj->state_in_sysfs = 0;`
				- Delete `kobj` from `kobj->kset`'s list.
				- `kobj->parent = NULL;`
		- [`int kobject_uevent_env(struct kobject *kobj, enum kobject_action action, char *envp_ext[])`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject_uevent.c#L457): Send an uenvet with environmental data.


### Normal Usage
#### Embedding kobjects
Usually kobject is embedded into other data structures as follows:
```c
struct mystruct {
	struct kobject kobj;
	int data;
};
```

You can get the pointer of instance containing the kobject by `container_of` as follows:
```c
struct kobject *kp;
struct mystruct *ptr;

ptr = container_of(kp, strcut mystruct, kobj);
```



## `struct kobj_type`
- It defines sysfs operations and default attributes for this `kobject`.

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L120](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L120)
```c
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;
	struct attribute **default_attrs;	/* use default_groups instead */
	const struct attribute_group **default_groups;
	const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
	const void *(*namespace)(struct kobject *kobj);
	void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

- Members
	- `void (*release)(struct kobject *kobj);`: This will be invoked when the refcount of this `kobject` equals to 0.
	- `struct attribuet **default_attrs;`: List of default attributes that this type of `kobject` should have.
	- `const struct sysfs_ops *sysfs_ops;`: Callbacks for sysfs attributes.

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L238](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/sysfs.h#L238)
```c
struct sysfs_ops {
	ssize_t	(*show)(struct kobject *, struct attribute *, char *);
	ssize_t	(*store)(struct kobject *, struct attribute *, const char *, size_t);
};
```

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


### Example of Dynamic Kobject
[https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L780](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L780)
```c
struct kobject *kobject_create_and_add(const char *name, struct kobject *parent)
{
/* snipped */
	kobject_init(kobj, &dynamic_kobj_ktype);
	return kobj;
}
```
- `kobject_create()` initializes `kobject` with `dynamic_kobj_ktype`.

[https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L764](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L764)
```c
static struct kobj_type dynamic_kobj_ktype = {
	.release	= dynamic_kobj_release,
	.sysfs_ops	= &kobj_sysfs_ops,
};
```

[https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L758](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L758)
```c
static void dynamic_kobj_release(struct kobject *kobj)
{
	pr_debug("kobject: (%p): %s\n", kobj, __func__);
	kfree(kobj);
}
```
- Just free the memory of kobject.



## `struct kset`
- It is a group of `kobject`s.
- `kset` contains its own `kobject`.

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L173](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L173)
```c
/**
 * struct kset - a set of kobjects of a specific type, belonging to a specific subsystem.
 *
 * A kset defines a group of kobjects.  They can be individually
 * different "types" but overall these kobjects all want to be grouped
 * together and operated on in the same manner.  ksets are used to
 * define the attribute callbacks and other common events that happen to
 * a kobject.
 *
 * @list: the list of all kobjects for this kset
 * @list_lock: a lock for iterating over the kobjects
 * @kobj: the embedded kobject for this kset (recursion, isn't it fun...)
 * @uevent_ops: the set of uevent operations for this kset.  These are
 * called whenever a kobject has something happen to it so that the kset
 * can add new environment variables, or filter out the uevents if so
 * desired.
 */
struct kset {
	struct list_head list;
	spinlock_t list_lock;
	struct kobject kobj;
	const struct kset_uevent_ops *uevent_ops;
} __randomize_layout;
```

- Description
	- `kset` defines a group of `kobject`s.
	- `kset` contains its own `kobject` internally and it can be treated the same way as a `kobject`.
	- `kset`s are always represented as sysfs directories.
- Members
	- `struct list_head list;`: Head of the list of `kobjects`.
	- `spinlock_t list_lock;`: Spinlock for `list`.
	- `struct kobject kobj;`: `kset` contains its own `kobject` internally.
	- `const struct kset_uevent_ops *uevent_ops;`: List of pointers for uevent-related functions allowing hotplug.
- Methods
	- [`kset_create_and_add()`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L1005)
		- [`kset_create()`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L962)
			- Allocate memory for a new kset by `kzalloc()`.
			- Set its name by `kobject_set_name()`.
			- Set `kset.uevent_ops` and `kset.kobj.parent` with the given arguments.
			- Set `kset.kobj.ktype = &kset_ktype;`.
			- Set `kset.kobj.kset = NULL;`.
		- [`kset_register()`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L870)
			- [`kset_init()`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L828)
				- Initialize `kobject` of the `kset` by `kobject_init_internal()`.
				- Initialize a list head (`kset.list`).
				- Initialize a spinlock of the list (`kset.list_lock`).
			- [`kobject_add_internal()`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L225)
				- Add itself into hierarchy.
			- [`kobject_uevent()`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject_uevent.c#L640)
				- Notify userspace by sending an uevent `KOBJ_ADD`.
	- [`kset_unregister()`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L890)
		- `kobject_del()`: delete `kset`'s `kobject`.
		- `kobject_put()`: increment refcount of `kset`'s `kobject`.
	- [`kset_get_ownership()`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L935)
	- [`kset_release()`](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L927)


[https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L941](https://elixir.bootlin.com/linux/v5.17.8/source/lib/kobject.c#L941)
```c
static struct kobj_type kset_ktype = {
	.sysfs_ops	= &kobj_sysfs_ops,
	.release	= kset_release,
	.get_ownership	= kset_get_ownership,
};
```

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L138](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L138)
```c
struct kset_uevent_ops {
	int (* const filter)(struct kobject *kobj);
	const char *(* const name)(struct kobject *kobj);
	int (* const uevent)(struct kobject *kobj, struct kobj_uevent_env *env);
};
```

- Members
	- `int (* const filter)(struct kobject *kobj);`: It is called before an uevent operation happens and determines if it's acceptable to generate an uevent for this kobject.
	- `const char *(* const name)(struct kobject *kobj);`: 
	- `int (* const uevent)(struct kobject *kobj, struct kobj_uevent_env *env);`: 


### Simple `kset` Hierarchy
```

                                        +-----------------------+
                                        | struct kset           <----------------------------------------------+
                                        +-----------------------+                                              |
                                        |+---------------------+|                                              |
                                        || struct kobject kobj |<------------------------------------------+   |
                                        |+---------------------+|                                          |   |
+---------------------------------------+ struct list_head list <--------------------------------------+   |   |
|                                       +-----------------------+                                      |   |   |
|                                                                                                      |   |   |
|                                                                                                      |   |   |
|  +------------------------+        +------------------------+           +------------------------+   |   |   |
|  | struct kobject         |        | struct kobject         |           | struct kobject         |   |   |   |
|  +------------------------+        +------------------------+           +------------------------+   |   |   |
+--> struct list_head entry +--------> struct list_head entry +--- ... ---> struct list_head entry +---+   |   |
   | struct kobject *parent +---+    | struct kobject *parent +---+       | struct kobject *parent +---+   |   |
   | struct kset *kset      +-+ |    | struct kset *kset      +-+ |       | struct kset *kset      +-+ |   |   |
   +------------------------+ | |    +------------------------+ | |       +------------------------+ | |   |   |
                              | +---------------------------------+------------------------------------+---+   |
                              |                                 |                                    |         |
                              +---------------------------------+----------------------------------------------+
```



## `struct kref`
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kref.h#L19](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kref.h#L19)
```c
struct kref {
	refcount_t refcount;
};
```

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/refcount.h#L113](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/refcount.h#L113)
```c
/**
 * typedef refcount_t - variant of atomic_t specialized for reference counts
 * @refs: atomic_t counter field
 *
 * The counter saturates at REFCOUNT_SATURATED and will not move once
 * there. This avoids wrapping the counter and causing 'spurious'
 * use-after-free bugs.
 */
typedef struct refcount_struct {
	atomic_t refs;
} refcount_t;
```

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/types.h#L168](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/types.h#L168)
```c
typedef struct {
	int counter;
} atomic_t;
```

- `struct kref` is a basic structure for reference counter of `struct kobject`.
- Methods
	- [`kref_init()`](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kref.h#L29)
		- Initialize.
		- As it turns out, `kref->refcount->refs->counter = 1`
	- [`kref_get()`](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kref.h#L43)
		- Increment refcount.
		- As it turns out, `kref->refcount->refs->counter += 1`
	- [`kref_put()`](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kref.h#L62)
		- Decrement refcount.
		- As it turns out, `kref->refcount->refs->counter -= 1`
		- If refcount equals to 0, it calls the given `release()` function.



## Links
- [kobject.rst - Documentation/core-api/kobject.rst - Linux source code (v5.17.9) - Bootlin](https://elixir.bootlin.com/linux/latest/source/Documentation/core-api/kobject.rst)
- [kref.rst - Documentation/core-api/kref.rst - Linux source code (v5.17.9) - Bootlin](https://elixir.bootlin.com/linux/latest/source/Documentation/core-api/kref.rst)
- [The zen of kobjects - LWN.net](https://lwn.net/Articles/51437/)
- [kobjects and sysfs - LWN.net](https://lwn.net/Articles/54651/)
