---
layout: post
title: kref
---



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
- [kref.rst - Documentation/core-api/kref.rst - Linux source code (v5.17.9) - Bootlin](https://elixir.bootlin.com/linux/latest/source/Documentation/core-api/kref.rst)
