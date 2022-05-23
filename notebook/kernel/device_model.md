---
layout: post
title: Device Model
---



## Device Model
- Prior to version 2.6, the kernel did not have a unified model for devices.
	- The traditional device model implemented some sort of tree-like structure (sometimes just a list) for the devices they control.
	- There wasn't any uniformity across the different bus types.
- The linux kernel device model is unification of all the disparate device models that were previously used in the kernel.
	- The current device model provides a common, uniform data model for describing a bus and devices that can appear under the bus.
	- The unified bus model includes a set of common attributes and a set of common callbacks.
	-  The kernel uses the following entities:
		- Device: A device that is attached to a bus. This might be a physical device, a part of physical device, and a virtual device.
		- Driver: A software entity that can be associated with a device and performs operations with it.
		- Bus: A device to which other devices can be attached.
		- Class: A type of device that has a similar behavior; there is a class of disks, partitions, serial ports, etc.



## Basics of Data Structures
- High-level
	- Public part
		- `struct bus_type` in [include/linux/device/bus.h](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/bus.h)
		- `struct class` in [include/linux/device/class.h](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/class.h)
		- `struct device_driver` in [include/linux/device/driver.h](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/driver.h)
		- `struct device` in [include/linux/device.h](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device.h#L486)
	- Private part
		- `struct subsys_private` in [drivers/base/base.h](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L40): Embedded in `struct bus_type` and `struct class`.
		- `struct driver_private` in [drivers/base/base.h](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L58): Embedded in `struct device_driver`.
		- `struct device_private` in [drivers/base/base.h](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L88): Embedded in `struct device_private`.
- Low-level: Please see [kobject](https://zulinx86.com/notebook/kernel/kobject) page for more details.
	- `struct kobject` in [include/linux/kobject.h](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L64)
	- `struct kobj_type` in [include/linux/kobject.h](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L120)
	- `struct kset` in [include/linux/kobject.h](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kobject.h#L173)
	- `struct kref` in [include/linux/kref.h](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/kref.h#L19)



## Initialization
- [`start_kernel()`](https://elixir.bootlin.com/linux/latest/source/init/main.c#L928)
	- [`vfs_caches_init()`](https://elixir.bootlin.com/linux/latest/source/fs/dcache.c#L3285)
		- [`mnt_init()`](https://elixir.bootlin.com/linux/latest/source/fs/namespace.c#L4359)
			- [`sysfs_init()`](https://elixir.bootlin.com/linux/latest/source/fs/sysfs/mount.c#L97): Initialize sysfs.
	- [`arch_call_rest_init()`](https://elixir.bootlin.com/linux/latest/source/init/main.c#L880)
		- [`rest_init()`](https://elixir.bootlin.com/linux/latest/source/init/main.c#L680)
			- [`kernel_init()`](https://elixir.bootlin.com/linux/latest/source/init/main.c#L1497)
				- [`kernel_init_freeable()`](https://elixir.bootlin.com/linux/latest/source/init/main.c#L1587)
					- [`do_basic_setup()`](https://elixir.bootlin.com/linux/latest/source/init/main.c#L1404)
						- [`driver_init()`](https://elixir.bootlin.com/linux/latest/source/drivers/base/init.c#L20)
							- [`devtmpfs_init()`](https://elixir.bootlin.com/linux/latest/source/drivers/base/devtmpfs.c#L458)
							- [`devices_init()`](https://elixir.bootlin.com/linux/latest/source/drivers/base/core.c#L3823)
								- Create `/sys/devices/` directory.
									- `struct kset *devices_kset = kset_create_and_add("devices", &device_uevent_ops, NULL);`
								- Create `/sys/dev/` directory.
									- `struct kobject *dev_kobj = kobject_create_and_add("dev", NULL);`
								- Create `/sys/block/` directory.
									- `struct kobject *sysfs_dev_block_kobj = kobject_create_and_add("block", dev_kobj);`
								- Create `/sys/char/` directory.
									- `struct kobject *sysfs_dev_char_kobj = kobject_create_and_add("char", dev_kobj);`
							- [`buses_init()`](https://elixir.bootlin.com/linux/latest/source/drivers/base/bus.c#L1167)
								- Create `/sys/bus/` directory.
									- `struct kset *bus_kset = kset_create_and_add("bus", &bus_uevent_ops, NULL);`
								- Create `/sys/devices/system/` directory.
									- `struct kset *system_kset = kset_create_and_add("system", NULL, &devices_kset->kobj);`
							- [`classes_init()`](https://elixir.bootlin.com/linux/latest/source/drivers/base/class.c#L573)
								- Create `/sys/class/` directory.
									- `struct kset *class_kset = kset_create_and_add("class", NULL, NULL);`
							- [`firmware_init()`](https://elixir.bootlin.com/linux/latest/source/drivers/base/firmware.c#L20)
								- Create `/sys/firmware/` directory.
									- `struct kobject *firmware_kobj = kobject_create_and_add("firmware", NULL);`
							- [`hypervisor_init()`](https://elixir.bootlin.com/linux/latest/source/drivers/base/hypervisor.c#L18)
								- Create `/sys/hypervisor/` directory.
									- `struct kobject *hypervisor_kobj = kobject_create_and_add("hypervisor", NULL);`




## `struct bus_type`
- General
	- A bus is a channel between the processor and one or more devices.
	- For the purposes of the device model, all devices are connected via a bus, even if it is a virtual one without a physical hardware correspondent.

[https://elixir.bootlin.com/linux/v5.17.7/source/include/linux/device/bus.h#L82](https://elixir.bootlin.com/linux/v5.17.7/source/include/linux/device/bus.h#L82)
```c
/**
 * struct bus_type - The bus type of the device
 *
 * @name:	The name of the bus.
 * @dev_name:	Used for subsystems to enumerate devices like ("foo%u", dev->id).
 * @dev_root:	Default device to use as the parent.
 * @bus_groups:	Default attributes of the bus.
 * @dev_groups:	Default attributes of the devices on the bus.
 * @drv_groups: Default attributes of the device drivers on the bus.
 * @match:	Called, perhaps multiple times, whenever a new device or driver
 *		is added for this bus. It should return a positive value if the
 *		given device can be handled by the given driver and zero
 *		otherwise. It may also return error code if determining that
 *		the driver supports the device is not possible. In case of
 *		-EPROBE_DEFER it will queue the device for deferred probing.
 * @uevent:	Called when a device is added, removed, or a few other things
 *		that generate uevents to add the environment variables.
 * @probe:	Called when a new device or driver add to this bus, and callback
 *		the specific driver's probe to initial the matched device.
 * @sync_state:	Called to sync device state to software state after all the
 *		state tracking consumers linked to this device (present at
 *		the time of late_initcall) have successfully bound to a
 *		driver. If the device has no consumers, this function will
 *		be called at late_initcall_sync level. If the device has
 *		consumers that are never bound to a driver, this function
 *		will never get called until they do.
 * @remove:	Called when a device removed from this bus.
 * @shutdown:	Called at shut-down time to quiesce the device.
 *
 * @online:	Called to put the device back online (after offlining it).
 * @offline:	Called to put the device offline for hot-removal. May fail.
 *
 * @suspend:	Called when a device on this bus wants to go to sleep mode.
 * @resume:	Called to bring a device on this bus out of sleep mode.
 * @num_vf:	Called to find out how many virtual functions a device on this
 *		bus supports.
 * @dma_configure:	Called to setup DMA configuration on a device on
 *			this bus.
 * @pm:		Power management operations of this bus, callback the specific
 *		device driver's pm-ops.
 * @iommu_ops:  IOMMU specific operations for this bus, used to attach IOMMU
 *              driver implementations to a bus and allow the driver to do
 *              bus-specific setup
 * @p:		The private data of the driver core, only the driver core can
 *		touch this.
 * @lock_key:	Lock class key for use by the lock validator
 * @need_parent_lock:	When probing or removing a device on this bus, the
 *			device core should lock the device's parent.
 *
 * A bus is a channel between the processor and one or more devices. For the
 * purposes of the device model, all devices are connected via a bus, even if
 * it is an internal, virtual, "platform" bus. Buses can plug into each other.
 * A USB controller is usually a PCI device, for example. The device model
 * represents the actual connections between buses and the devices they control.
 * A bus is represented by the bus_type structure. It contains the name, the
 * default attributes, the bus' methods, PM operations, and the driver core's
 * private data.
 */
struct bus_type {
	const char		*name;
	const char		*dev_name;
	struct device		*dev_root;
	const struct attribute_group **bus_groups;
	const struct attribute_group **dev_groups;
	const struct attribute_group **drv_groups;

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
	void (*sync_state)(struct device *dev);
	void (*remove)(struct device *dev);
	void (*shutdown)(struct device *dev);

	int (*online)(struct device *dev);
	int (*offline)(struct device *dev);

	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);

	int (*num_vf)(struct device *dev);

	int (*dma_configure)(struct device *dev);

	const struct dev_pm_ops *pm;

	const struct iommu_ops *iommu_ops;

	struct subsys_private *p;
	struct lock_class_key lock_key;

	bool need_parent_lock;
};
```

[https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L40](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L40)
```c
/**
 * struct subsys_private - structure to hold the private to the driver core portions of the bus_type/class structure.
 *
 * @subsys - the struct kset that defines this subsystem
 * @devices_kset - the subsystem's 'devices' directory
 * @interfaces - list of subsystem interfaces associated
 * @mutex - protect the devices, and interfaces lists.
 *
 * @drivers_kset - the list of drivers associated
 * @klist_devices - the klist to iterate over the @devices_kset
 * @klist_drivers - the klist to iterate over the @drivers_kset
 * @bus_notifier - the bus notifier list for anything that cares about things
 *                 on this bus.
 * @bus - pointer back to the struct bus_type that this structure is associated
 *        with.
 *
 * @glue_dirs - "glue" directory to put in-between the parent device to
 *              avoid namespace conflicts
 * @class - pointer back to the struct class that this structure is associated
 *          with.
 *
 * This structure is the one that is the actual kobject allowing struct
 * bus_type/class to be statically allocated safely.  Nothing outside of the
 * driver core should ever touch these fields.
 */
struct subsys_private {
	struct kset subsys;
	struct kset *devices_kset;
	struct list_head interfaces;
	struct mutex mutex;

	struct kset *drivers_kset;
	struct klist klist_devices;
	struct klist klist_drivers;
	struct blocking_notifier_head bus_notifier;
	unsigned int drivers_autoprobe:1;
	struct bus_type *bus;

	struct kset glue_dirs;
	struct class *class;
};
```

- Members
- Methods
	- [`int buses_init(void)`](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/bus.c#L1167)
		- Initialize `bus_kset` variable and create `/sys/bus/` directory.
		- Initialize `system_kset` variable and create `/sys/devices/system/` directory.

[https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/bus.c#L161](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/bus.c#L161)
```c
static struct kobj_type bus_ktype = {
	.sysfs_ops	= &bus_sysfs_ops,
	.release	= bus_release,
};
```
```c
static const struct sysfs_ops bus_sysfs_ops = {
	.show	= bus_attr_show,
	.store	= bus_attr_store,
};
```
```c
/*
 * sysfs bindings for buses
 */
static ssize_t bus_attr_show(struct kobject *kobj, struct attribute *attr,
			     char *buf)
{
	struct bus_attribute *bus_attr = to_bus_attr(attr);
	struct subsys_private *subsys_priv = to_subsys_private(kobj);
	ssize_t ret = 0;

	if (bus_attr->show)
		ret = bus_attr->show(subsys_priv->bus, buf);
	return ret;
}

static ssize_t bus_attr_store(struct kobject *kobj, struct attribute *attr,
			      const char *buf, size_t count)
{
	struct bus_attribute *bus_attr = to_bus_attr(attr);
	struct subsys_private *subsys_priv = to_subsys_private(kobj);
	ssize_t ret = 0;

	if (bus_attr->store)
		ret = bus_attr->store(subsys_priv->bus, buf, count);
	return ret;
}
```
```c
static void bus_release(struct kobject *kobj)
{
	struct subsys_private *priv = to_subsys_private(kobj);
	struct bus_type *bus = priv->bus;

	kfree(priv);
	bus->p = NULL;
}
```

[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/bus.h#L123](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/bus.h#L123)
```c
struct bus_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct bus_type *bus, char *buf);
	ssize_t (*store)(struct bus_type *bus, const char *buf, size_t count);
};
```

[https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/bus.c#L175](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/bus.c#L175)
```c
static const struct kset_uevent_ops bus_uevent_ops = {
	.filter = bus_uevent_filter,
};
```
```c
static int bus_uevent_filter(struct kobject *kobj)
{
	const struct kobj_type *ktype = get_ktype(kobj);

	if (ktype == &bus_ktype)
		return 1;
	return 0;
}
```



## `struct class`
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/class.h#L54](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device/class.h#L54)
```c
/**
 * struct class - device classes
 * @name:	Name of the class.
 * @owner:	The module owner.
 * @class_groups: Default attributes of this class.
 * @dev_groups:	Default attributes of the devices that belong to the class.
 * @dev_kobj:	The kobject that represents this class and links it into the hierarchy.
 * @dev_uevent:	Called when a device is added, removed from this class, or a
 *		few other things that generate uevents to add the environment
 *		variables.
 * @devnode:	Callback to provide the devtmpfs.
 * @class_release: Called to release this class.
 * @dev_release: Called to release the device.
 * @shutdown_pre: Called at shut-down time before driver shutdown.
 * @ns_type:	Callbacks so sysfs can detemine namespaces.
 * @namespace:	Namespace of the device belongs to this class.
 * @get_ownership: Allows class to specify uid/gid of the sysfs directories
 *		for the devices belonging to the class. Usually tied to
 *		device's namespace.
 * @pm:		The default device power management operations of this class.
 * @p:		The private data of the driver core, no one other than the
 *		driver core can touch this.
 *
 * A class is a higher-level view of a device that abstracts out low-level
 * implementation details. Drivers may see a SCSI disk or an ATA disk, but,
 * at the class level, they are all simply disks. Classes allow user space
 * to work with devices based on what they do, rather than how they are
 * connected or how they work.
 */
struct class {
	const char		*name;
	struct module		*owner;

	const struct attribute_group	**class_groups;
	const struct attribute_group	**dev_groups;
	struct kobject			*dev_kobj;

	int (*dev_uevent)(struct device *dev, struct kobj_uevent_env *env);
	char *(*devnode)(struct device *dev, umode_t *mode);

	void (*class_release)(struct class *class);
	void (*dev_release)(struct device *dev);

	int (*shutdown_pre)(struct device *dev);

	const struct kobj_ns_type_operations *ns_type;
	const void *(*namespace)(struct device *dev);

	void (*get_ownership)(struct device *dev, kuid_t *uid, kgid_t *gid);

	const struct dev_pm_ops *pm;

	struct subsys_private *p;
};
```

[https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L40](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L40)
```c
/**
 * struct subsys_private - structure to hold the private to the driver core portions of the bus_type/class structure.
 *
 * @subsys - the struct kset that defines this subsystem
 * @devices_kset - the subsystem's 'devices' directory
 * @interfaces - list of subsystem interfaces associated
 * @mutex - protect the devices, and interfaces lists.
 *
 * @drivers_kset - the list of drivers associated
 * @klist_devices - the klist to iterate over the @devices_kset
 * @klist_drivers - the klist to iterate over the @drivers_kset
 * @bus_notifier - the bus notifier list for anything that cares about things
 *                 on this bus.
 * @bus - pointer back to the struct bus_type that this structure is associated
 *        with.
 *
 * @glue_dirs - "glue" directory to put in-between the parent device to
 *              avoid namespace conflicts
 * @class - pointer back to the struct class that this structure is associated
 *          with.
 *
 * This structure is the one that is the actual kobject allowing struct
 * bus_type/class to be statically allocated safely.  Nothing outside of the
 * driver core should ever touch these fields.
 */
struct subsys_private {
	struct kset subsys;
	struct kset *devices_kset;
	struct list_head interfaces;
	struct mutex mutex;

	struct kset *drivers_kset;
	struct klist klist_devices;
	struct klist klist_drivers;
	struct blocking_notifier_head bus_notifier;
	unsigned int drivers_autoprobe:1;
	struct bus_type *bus;

	struct kset glue_dirs;
	struct class *class;
};
```

- Members
- Methods
	- [`init classes_init(void)`](https://elixir.bootlin.com/linux/latest/source/drivers/base/class.c#L573)
		- Initialize `class_kset` variable and create `/sys/class/` directory.


[https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/class.c#L78](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/class.c#L78)
```c
static struct kobj_type class_ktype = {
	.sysfs_ops	= &class_sysfs_ops,
	.release	= class_release,
	.child_ns_type	= class_child_ns_type,
};
```
```c
static const struct sysfs_ops class_sysfs_ops = {
	.show	   = class_attr_show,
	.store	   = class_attr_store,
};
```
```c
static ssize_t class_attr_show(struct kobject *kobj, struct attribute *attr,
			       char *buf)
{
	struct class_attribute *class_attr = to_class_attr(attr);
	struct subsys_private *cp = to_subsys_private(kobj);
	ssize_t ret = -EIO;

	if (class_attr->show)
		ret = class_attr->show(cp->class, class_attr, buf);
	return ret;
}

static ssize_t class_attr_store(struct kobject *kobj, struct attribute *attr,
				const char *buf, size_t count)
{
	struct class_attribute *class_attr = to_class_attr(attr);
	struct subsys_private *cp = to_subsys_private(kobj);
	ssize_t ret = -EIO;

	if (class_attr->store)
		ret = class_attr->store(cp->class, class_attr, buf, count);
	return ret;
}

static void class_release(struct kobject *kobj)
{
	struct subsys_private *cp = to_subsys_private(kobj);
	struct class *class = cp->class;

	pr_debug("class '%s': release.\n", class->name);

	if (class->class_release)
		class->class_release(class);
	else
		pr_debug("class '%s' does not have a release() function, "
			 "be careful\n", class->name);

	kfree(cp);
}

static const struct kobj_ns_type_operations *class_child_ns_type(struct kobject *kobj)
{
	struct subsys_private *cp = to_subsys_private(kobj);
	struct class *class = cp->class;

	return class->ns_type;
}
```

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



## `struct device_driver`
[https://elixir.bootlin.com/linux/v5.17.7/source/include/linux/device/driver.h#L96](https://elixir.bootlin.com/linux/v5.17.7/source/include/linux/device/driver.h#L96)
```c
/**
 * struct device_driver - The basic device driver structure
 * @name:	Name of the device driver.
 * @bus:	The bus which the device of this driver belongs to.
 * @owner:	The module owner.
 * @mod_name:	Used for built-in modules.
 * @suppress_bind_attrs: Disables bind/unbind via sysfs.
 * @probe_type:	Type of the probe (synchronous or asynchronous) to use.
 * @of_match_table: The open firmware table.
 * @acpi_match_table: The ACPI match table.
 * @probe:	Called to query the existence of a specific device,
 *		whether this driver can work with it, and bind the driver
 *		to a specific device.
 * @sync_state:	Called to sync device state to software state after all the
 *		state tracking consumers linked to this device (present at
 *		the time of late_initcall) have successfully bound to a
 *		driver. If the device has no consumers, this function will
 *		be called at late_initcall_sync level. If the device has
 *		consumers that are never bound to a driver, this function
 *		will never get called until they do.
 * @remove:	Called when the device is removed from the system to
 *		unbind a device from this driver.
 * @shutdown:	Called at shut-down time to quiesce the device.
 * @suspend:	Called to put the device to sleep mode. Usually to a
 *		low power state.
 * @resume:	Called to bring a device from sleep mode.
 * @groups:	Default attributes that get created by the driver core
 *		automatically.
 * @dev_groups:	Additional attributes attached to device instance once
 *		it is bound to the driver.
 * @pm:		Power management operations of the device which matched
 *		this driver.
 * @coredump:	Called when sysfs entry is written to. The device driver
 *		is expected to call the dev_coredump API resulting in a
 *		uevent.
 * @p:		Driver core's private data, no one other than the driver
 *		core can touch this.
 *
 * The device driver-model tracks all of the drivers known to the system.
 * The main reason for this tracking is to enable the driver core to match
 * up drivers with new devices. Once drivers are known objects within the
 * system, however, a number of other things become possible. Device drivers
 * can export information and configuration variables that are independent
 * of any specific device.
 */
struct device_driver {
	const char		*name;
	struct bus_type		*bus;

	struct module		*owner;
	const char		*mod_name;	/* used for built-in modules */

	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */
	enum probe_type probe_type;

	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;

	int (*probe) (struct device *dev);
	void (*sync_state)(struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;
	const struct attribute_group **dev_groups;

	const struct dev_pm_ops *pm;
	void (*coredump) (struct device *dev);

	struct driver_private *p;
};
```

[https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L58](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L58)
```c
struct driver_private {
	struct kobject kobj;
	struct klist klist_devices;
	struct klist_node knode_bus;
	struct module_kobject *mkobj;
	struct device_driver *driver;
};
```

- Members
- Methods

[https://elixir.bootlin.com/linux/v5.17.7/source/drivers/base/bus.c#L94](https://elixir.bootlin.com/linux/v5.17.7/source/drivers/base/bus.c#L94)
```c
static struct kobj_type driver_ktype = {
	.sysfs_ops	= &driver_sysfs_ops,
	.release	= driver_release,
};
```
```c
static const struct sysfs_ops driver_sysfs_ops = {
	.show	= drv_attr_show,
	.store	= drv_attr_store,
};
```
```c
static ssize_t drv_attr_show(struct kobject *kobj, struct attribute *attr,
			     char *buf)
{
	struct driver_attribute *drv_attr = to_drv_attr(attr);
	struct driver_private *drv_priv = to_driver(kobj);
	ssize_t ret = -EIO;

	if (drv_attr->show)
		ret = drv_attr->show(drv_priv->driver, buf);
	return ret;
}

static ssize_t drv_attr_store(struct kobject *kobj, struct attribute *attr,
			      const char *buf, size_t count)
{
	struct driver_attribute *drv_attr = to_drv_attr(attr);
	struct driver_private *drv_priv = to_driver(kobj);
	ssize_t ret = -EIO;

	if (drv_attr->store)
		ret = drv_attr->store(drv_priv->driver, buf, count);
	return ret;
}
```
```c
static void driver_release(struct kobject *kobj)
{
	struct driver_private *drv_priv = to_driver(kobj);

	pr_debug("driver: '%s': %s\n", kobject_name(kobj), __func__);
	kfree(drv_priv);
}
```



## `struct device`
[https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device.h#L486](https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/device.h#L486)
```c
/**
 * struct device - The basic device structure
 * @parent:	The device's "parent" device, the device to which it is attached.
 * 		In most cases, a parent device is some sort of bus or host
 * 		controller. If parent is NULL, the device, is a top-level device,
 * 		which is not usually what you want.
 * @p:		Holds the private data of the driver core portions of the device.
 * 		See the comment of the struct device_private for detail.
 * @kobj:	A top-level, abstract class from which other classes are derived.
 * @init_name:	Initial name of the device.
 * @type:	The type of device.
 * 		This identifies the device type and carries type-specific
 * 		information.
 * @mutex:	Mutex to synchronize calls to its driver.
 * @lockdep_mutex: An optional debug lock that a subsystem can use as a
 * 		peer lock to gain localized lockdep coverage of the device_lock.
 * @bus:	Type of bus device is on.
 * @driver:	Which driver has allocated this
 * @platform_data: Platform data specific to the device.
 * 		Example: For devices on custom boards, as typical of embedded
 * 		and SOC based hardware, Linux often uses platform_data to point
 * 		to board-specific structures describing devices and how they
 * 		are wired.  That can include what ports are available, chip
 * 		variants, which GPIO pins act in what additional roles, and so
 * 		on.  This shrinks the "Board Support Packages" (BSPs) and
 * 		minimizes board-specific #ifdefs in drivers.
 * @driver_data: Private pointer for driver specific info.
 * @links:	Links to suppliers and consumers of this device.
 * @power:	For device power management.
 *		See Documentation/driver-api/pm/devices.rst for details.
 * @pm_domain:	Provide callbacks that are executed during system suspend,
 * 		hibernation, system resume and during runtime PM transitions
 * 		along with subsystem-level and driver-level callbacks.
 * @em_pd:	device's energy model performance domain
 * @pins:	For device pin management.
 *		See Documentation/driver-api/pin-control.rst for details.
 * @msi:	MSI related data
 * @numa_node:	NUMA node this device is close to.
 * @dma_ops:    DMA mapping operations for this device.
 * @dma_mask:	Dma mask (if dma'ble device).
 * @coherent_dma_mask: Like dma_mask, but for alloc_coherent mapping as not all
 * 		hardware supports 64-bit addresses for consistent allocations
 * 		such descriptors.
 * @bus_dma_limit: Limit of an upstream bridge or bus which imposes a smaller
 *		DMA limit than the device itself supports.
 * @dma_range_map: map for DMA memory ranges relative to that of RAM
 * @dma_parms:	A low level driver may set these to teach IOMMU code about
 * 		segment limitations.
 * @dma_pools:	Dma pools (if dma'ble device).
 * @dma_mem:	Internal for coherent mem override.
 * @cma_area:	Contiguous memory area for dma allocations
 * @dma_io_tlb_mem: Pointer to the swiotlb pool used.  Not for driver use.
 * @archdata:	For arch-specific additions.
 * @of_node:	Associated device tree node.
 * @fwnode:	Associated device node supplied by platform firmware.
 * @devt:	For creating the sysfs "dev".
 * @id:		device instance
 * @devres_lock: Spinlock to protect the resource of the device.
 * @devres_head: The resources list of the device.
 * @knode_class: The node used to add the device to the class list.
 * @class:	The class of the device.
 * @groups:	Optional attribute groups.
 * @release:	Callback to free the device after all references have
 * 		gone away. This should be set by the allocator of the
 * 		device (i.e. the bus driver that discovered the device).
 * @iommu_group: IOMMU group the device belongs to.
 * @iommu:	Per device generic IOMMU runtime data
 * @removable:  Whether the device can be removed from the system. This
 *              should be set by the subsystem / bus driver that discovered
 *              the device.
 *
 * @offline_disabled: If set, the device is permanently online.
 * @offline:	Set after successful invocation of bus type's .offline().
 * @of_node_reused: Set if the device-tree node is shared with an ancestor
 *              device.
 * @state_synced: The hardware state of this device has been synced to match
 *		  the software state of this device by calling the driver/bus
 *		  sync_state() callback.
 * @can_match:	The device has matched with a driver at least once or it is in
 *		a bus (like AMBA) which can't check for matching drivers until
 *		other devices probe successfully.
 * @dma_coherent: this particular device is dma coherent, even if the
 *		architecture supports non-coherent devices.
 * @dma_ops_bypass: If set to %true then the dma_ops are bypassed for the
 *		streaming DMA operations (->map_* / ->unmap_* / ->sync_*),
 *		and optionall (if the coherent mask is large enough) also
 *		for dma allocations.  This flag is managed by the dma ops
 *		instance from ->dma_supported.
 *
 * At the lowest level, every device in a Linux system is represented by an
 * instance of struct device. The device structure contains the information
 * that the device model core needs to model the system. Most subsystems,
 * however, track additional information about the devices they host. As a
 * result, it is rare for devices to be represented by bare device structures;
 * instead, that structure, like kobject structures, is usually embedded within
 * a higher-level representation of the device.
 */
struct device {
	struct kobject kobj;
	struct device		*parent;

	struct device_private	*p;

	const char		*init_name; /* initial name of the device */
	const struct device_type *type;

	struct bus_type	*bus;		/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this
					   device */
	void		*platform_data;	/* Platform specific data, device
					   core doesn't touch it */
	void		*driver_data;	/* Driver data, set and get with
					   dev_set_drvdata/dev_get_drvdata */
#ifdef CONFIG_PROVE_LOCKING
	struct mutex		lockdep_mutex;
#endif
	struct mutex		mutex;	/* mutex to synchronize calls to
					 * its driver.
					 */

	struct dev_links_info	links;
	struct dev_pm_info	power;
	struct dev_pm_domain	*pm_domain;

#ifdef CONFIG_ENERGY_MODEL
	struct em_perf_domain	*em_pd;
#endif

#ifdef CONFIG_PINCTRL
	struct dev_pin_info	*pins;
#endif
	struct dev_msi_info	msi;
#ifdef CONFIG_DMA_OPS
	const struct dma_map_ops *dma_ops;
#endif
	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask;/* Like dma_mask, but for
					     alloc_coherent mappings as
					     not all hardware supports
					     64 bit addresses for consistent
					     allocations such descriptors. */
	u64		bus_dma_limit;	/* upstream dma constraint */
	const struct bus_dma_region *dma_range_map;

	struct device_dma_parameters *dma_parms;

	struct list_head	dma_pools;	/* dma pools (if dma'ble) */

#ifdef CONFIG_DMA_DECLARE_COHERENT
	struct dma_coherent_mem	*dma_mem; /* internal for coherent mem
					     override */
#endif
#ifdef CONFIG_DMA_CMA
	struct cma *cma_area;		/* contiguous memory area for dma
					   allocations */
#endif
#ifdef CONFIG_SWIOTLB
	struct io_tlb_mem *dma_io_tlb_mem;
#endif
	/* arch specific additions */
	struct dev_archdata	archdata;

	struct device_node	*of_node; /* associated device tree node */
	struct fwnode_handle	*fwnode; /* firmware device node */

#ifdef CONFIG_NUMA
	int		numa_node;	/* NUMA node this device is close to */
#endif
	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	u32			id;	/* device instance */

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	struct class		*class;
	const struct attribute_group **groups;	/* optional groups */

	void	(*release)(struct device *dev);
	struct iommu_group	*iommu_group;
	struct dev_iommu	*iommu;

	enum device_removable	removable;

	bool			offline_disabled:1;
	bool			offline:1;
	bool			of_node_reused:1;
	bool			state_synced:1;
	bool			can_match:1;
#if defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_DEVICE) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU_ALL)
	bool			dma_coherent:1;
#endif
#ifdef CONFIG_DMA_OPS_BYPASS
	bool			dma_ops_bypass : 1;
#endif
};
```

[https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L88](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/base.h#L88)
```c
/**
 * struct device_private - structure to hold the private to the driver core portions of the device structure.
 *
 * @klist_children - klist containing all children of this device
 * @knode_parent - node in sibling list
 * @knode_driver - node in driver list
 * @knode_bus - node in bus list
 * @knode_class - node in class list
 * @deferred_probe - entry in deferred_probe_list which is used to retry the
 *	binding of drivers which were unable to get all the resources needed by
 *	the device; typically because it depends on another driver getting
 *	probed first.
 * @async_driver - pointer to device driver awaiting probe via async_probe
 * @device - pointer back to the struct device that this structure is
 * associated with.
 * @dead - This device is currently either in the process of or has been
 *	removed from the system. Any asynchronous events scheduled for this
 *	device should exit without taking any action.
 *
 * Nothing outside of the driver core should ever touch these fields.
 */
struct device_private {
	struct klist klist_children;
	struct klist_node knode_parent;
	struct klist_node knode_driver;
	struct klist_node knode_bus;
	struct klist_node knode_class;
	struct list_head deferred_probe;
	struct device_driver *async_driver;
	char *deferred_probe_reason;
	struct device *device;
	u8 dead:1;
};
```

- Members
- Methods
	- [`devices_init()`](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/core.c#L3823)
		- Initialzie `devices_kset` variable and create `/sys/devices/` directory.

[https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/core.c#L2255](https://elixir.bootlin.com/linux/v5.17.8/source/drivers/base/core.c#L2255)
```c
static struct kobj_type device_ktype = {
	.release	= device_release,
	.sysfs_ops	= &dev_sysfs_ops,
	.namespace	= device_namespace,
	.get_ownership	= device_get_ownership,
};
```
```c
static const struct sysfs_ops dev_sysfs_ops = {
	.show	= dev_attr_show,
	.store	= dev_attr_store,
};
```
```c
/**
 * device_release - free device structure.
 * @kobj: device's kobject.
 *
 * This is called once the reference count for the object
 * reaches 0. We forward the call to the device's release
 * method, which should handle actually freeing the structure.
 */
static void device_release(struct kobject *kobj)
{
	struct device *dev = kobj_to_dev(kobj);
	struct device_private *p = dev->p;

	/*
	 * Some platform devices are driven without driver attached
	 * and managed resources may have been acquired.  Make sure
	 * all resources are released.
	 *
	 * Drivers still can add resources into device after device
	 * is deleted but alive, so release devres here to avoid
	 * possible memory leak.
	 */
	devres_release_all(dev);

	kfree(dev->dma_range_map);

	if (dev->release)
		dev->release(dev);
	else if (dev->type && dev->type->release)
		dev->type->release(dev);
	else if (dev->class && dev->class->dev_release)
		dev->class->dev_release(dev);
	else
		WARN(1, KERN_ERR "Device '%s' does not have a release() function, it is broken and must be fixed. See Documentation/core-api/kobject.rst.\n",
			dev_name(dev));
	kfree(p);
}

static const void *device_namespace(struct kobject *kobj)
{
	struct device *dev = kobj_to_dev(kobj);
	const void *ns = NULL;

	if (dev->class && dev->class->ns_type)
		ns = dev->class->namespace(dev);

	return ns;
}

static void device_get_ownership(struct kobject *kobj, kuid_t *uid, kgid_t *gid)
{
	struct device *dev = kobj_to_dev(kobj);

	if (dev->class && dev->class->get_ownership)
		dev->class->get_ownership(dev, uid, gid);
}
```
```c
static ssize_t dev_attr_show(struct kobject *kobj, struct attribute *attr,
			     char *buf)
{
	struct device_attribute *dev_attr = to_dev_attr(attr);
	struct device *dev = kobj_to_dev(kobj);
	ssize_t ret = -EIO;

	if (dev_attr->show)
		ret = dev_attr->show(dev, dev_attr, buf);
	if (ret >= (ssize_t)PAGE_SIZE) {
		printk("dev_attr_show: %pS returned bad count\n",
				dev_attr->show);
	}
	return ret;
}

static ssize_t dev_attr_store(struct kobject *kobj, struct attribute *attr,
			      const char *buf, size_t count)
{
	struct device_attribute *dev_attr = to_dev_attr(attr);
	struct device *dev = kobj_to_dev(kobj);
	ssize_t ret = -EIO;

	if (dev_attr->store)
		ret = dev_attr->store(dev, dev_attr, buf, count);
	return ret;
}
```



## Links
- [Driver porting: Device model overview - LWN.net](https://lwn.net/Articles/31185/)
- [A fresh look at the kernel's device model - LWN.net](https://lwn.net/Articles/645810/)
- [A tour of /sys/devices - LWN.net](https://lwn.net/Articles/646617/)
- [14.4. Buses, Devices, and Drivers](http://www.makelinux.net/ldd3/chp-14-sect-4.shtml)
- [Driver Model — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/driver-api/driver-model/index.html)
- [Chapter 14. The Linux Device Model](http://www.makelinux.net/ldd3/chp-14.shtml)
