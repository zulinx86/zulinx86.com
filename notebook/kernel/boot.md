---
layout: post
title: Boot Sequence
---


## Boot Sequence on x86
1. Power On
1. BIOS (Legacy BIOS / UEFI)
1. Bootloader (LILO / GRUB Legacy / GRUB2)
1. Kernel
1. Service Manager



## Power On
1. CPU receives the reset signal.
1. CPU initializes its registers and set its mode to real mode (16-bit mode).
1. CPU starts with the reset vector (0xFFFFFFF0).
	- IP = 0xFFFF0.
	- CS selector = 0xF0000.
	- CS base address = 0xFFFF0000 (until reload CS).
1. The reset vector contains a jump instruction to the BIOS.



## BIOS (Basic Input/Output System)
### Legacy BIOS
- How it works
	1. Legacy BIOS performs POST (Power-On Self Test) to initialize and test hardwares.
	1. It detects bootable disks whose first sector is ended with the magic number (`0x55 0xAA`).
	1. It loads the first sector into memory address 0x7C00 and jumps to it. It contains the very first part of bootloader.
- BIOS Services
	- Legancy BIOS provides BIOS services related to display, keyboard, disk and so on, which make it easy to do boot-up tasks.
	- The memory below address 0x00400 contains the interrupt vector table (IVT).
	- Please see [BIOS](https://zulinx86.com/notebook/lowlayer/bios) page for more details.


### UEFI (Unified Extensible Firmware Interface)
- How it works
	1. UEFI firmware performs POST same as legacy BIOS.
	1. It also enables the A20 gate and prepares either protected mode environment with flat segmentation (for 32-bit x86 processors) or a long mode environment with identity-mapped paging (for 64-bit x86 processors).
	1. It loads a bootloader as an EFI application (relocatable PE executable file) from an EFI System Partition (ESP) on a GPT-partitioned boot device.
	1. It calls the bootloader's main entry point.
- UEFI Services
	- List of UEFI services
		- UEFI boot services
		- UEFI runtime services
		- Protocol services
	- UEFI firmware passes UEFI system table, which contains pointers to UEFI boot services and UEFI runtime services.
	- Protocol services are groups of related functions and data fields that are named by a Globally Unique Identifier (GUID). Protocol services are typically used to provide software abstraction for devices such as consoles, mass storage devices and networks.
- UEFI can pass its control to the bootloader in protected mode (32-bit mode) or long mode (64-bit mode), while only real mode (16-bit mode) in legacy BIOS.

#### Samples
```
$ sudo parted -l
Model: ATA Micron_1100_MTFD (scsi)
Disk /dev/sda: 256GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End    File system  Name                  Flags
 1      17.4kB  538MB  fat32        EFI System Partition  boot, esp
 2      538MB   256GB  ext4
```
```
$ lsblk | grep sda
sda      8:0    0 238.5G  0 disk
├─sda1   8:1    0 513.1M  0 part /boot/efi
└─sda2   8:2    0   238G  0 part /
```
```
$ sudo tree /boot/efi
/boot/efi
└── EFI
    ├── BOOT
    │   ├── BOOTX64.EFI
    │   ├── fbx64.efi
    │   └── mmx64.efi
    └── ubuntu
        ├── BOOTX64.EFI
        ├── grub.cfg
        ├── grubx64.efi
        ├── mmx64.efi
        └── shimx64.efi

3 directories, 8 files
```
```
$ efibootmgr -v
BootCurrent: 0005
Timeout: 1 seconds
BootOrder: 0005
Boot0005* ubuntu	HD(1,GPT,53ac790e-858e-4019-ba43-e736fed859d6,0x22,0x10089e)/File(\EFI\UBUNTU\SHIMX64.EFI)
```
- `shimx64.efi` is a first stage UEFI bootloader which enables secure boot.
- `shimx64.efi` will load `grubx64.efi` which is the GRUB2 bootloader.



## Bootloader
- There are several types of bootloaders such as LILO, Syslinux, GRUB Legacy, GRUB2. GRUB2 is often used recently.
- GRUB stands for GRand Unified Bootloader.


### GRUB Legacy
- GRUB legacy does not support UEFI and GPT.
- How it works
	1. Stage 1
		- Stage 1 is located in the first sector of the boot disk and is loaded by BIOS.
		- Stage 1 is responsible for just loading stage 1.5, since its space is too small.
	1. Stage 1.5
		- Stage 1.5 is stored in the first 30 KiB of the boot disk immediately after the first sector and before the first MBR partition.
		- This space is not available in GPT since GPT partition
		- Stage 1.5 includes filesystem drivers, which enables to load stage 2 located in a filesystem.
	1. Stage 2
		- Stage 2 consists of a number of components:
			- A command processor: which can execute a number of commands associated with loading OS components.
			- A menu processor: which reads meny entries from the configuration file (`/boot/grub/menu.lst`).
			- A display system: which can display menus, and other information on the screen and get input from the keyboard.
- Commands
	- `grub-install <path>`: Install GRUB legacy.
- Configuration Files
	- `/boot/grub/menu.lst`
		- `default <num>`: Default entry.
		- `timeout <sec>`: Timeout for entry selection.
		- `splashimage <path>`: Path to the splash image.
		- For each menu entry
			- `title <name>`: Name of OS.
			- `root (<disk>,<part>)`: Disk and partition for loading OS.
			- `kernel <path> <options>`: Kernel image path and options.
			- `initrd <path>`: Initial RAM disk path.


### GRUB2
- How it works
	- Legacy BIOS
		1. `boot.img`
			- `boot.img` is the first sector of the boot disk and is responsible for loading the first sector (`diskboot.img`) of `core.img`.
		1. `core.img`
			- `core.img` contains necessary modules to access `/boot/grub` or `/boot/grub2/`.
			- It reads the configuration file (`/boot/grub/grub.cfg` or `/boot/grub2/grub.cfg`) and loads the kernel image and the initial RAM disk based on it.
			- MBR (Master Boot Record) partitioned disk
				- core.img is located immediately after the first sector and before the first partition.
			- GPT (GUID Partition Table) partitioned disk
				- core.img is located in BIOS boot partition because there are no empty space after the first sector.
	- UEFI
		- `/boot/efi/EFI/<dist>/grubx64.efi` is loaded by UEFI firmware.
		- `grubx64.efi` reads the configuration file `/boot/efi/EFI/<dist>/grub.cfg` from the ESP and loads the kernel image and the initial RAM disk based on it.
- Commands
	- `grub2-install <path>`: Install GRUB2.
	- `grub-mkconfig -o <path>` / `grub2-mkconfig -o <path>`: Generate `/boot/grub/grub.cfg` / `/boot/grub2/grub.cfg` based on `/etc/default/grub` and `/etc/grub.d/*`.
- Configuration Files
	- `/etc/default/grub`: Custom settings.
		- `GRUB_DEFAULT=<num>`: Default menu entry.
		- `GRUB_TIMEOUT=<sec>`: Timeout for displaying menu.
		- `GRUB_DISABLE_RECOVERY="<true/false>"`: Recovery mode disabled or not.
			- Unless `GRUB_DISABLE_RECOVERY` is set to true, two menu entries will be generated for each Linux kernel: one default entry and one entry for recovery mode.
		- `GRUB_CMDLINE_LINUX="<options>"`: Command line arguments to add to menu entries.
		- `GRUB_CMDLINE_LINUX_DEFAULT="<options>"`: Command line arguments to add only to the default menu entry.
	- `/etc/grub.d/*`: Scripts to build the menu.



## Kernel (Under Construction)
- How it works
	1. The kernel goes to 64-bit mode (long mode).
	1. It decompresses itself first and does some initialization.
	1. It spawns up the init process (PID = 1).



## Service Manager (Under Construction)
### SysVinit


### Upstart


### Systemd



## Source Code
### GRUB2
#### grub-core/boot/i386/pc/boot.S
- `boot.img` is the first sector of the boot disk.
- It starts in 16-bit mode and is loaded into 0x7C00.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L121](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L121)
```
	/* Tell GAS to generate 16-bit instructions so that this code works
	   in real mode. */
	.code16

.globl _start, start;
_start:
start:
	/*
	 * _start is loaded at 0x7c00 and is jumped to with CS:IP 0:0x7c00
	 */
```

- It ends with 0xAA55.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L539](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L539)
```
	.org GRUB_BOOT_MACHINE_PART_END
	
/* the last 2 bytes in the sector 0 contain the signature */
	.word	GRUB_BOOT_MACHINE_SIGNATURE
```

[https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/pc/boot.h#L24](https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/pc/boot.h#L24)
```c
/* The signature for bootloader.  */
#define GRUB_BOOT_MACHINE_SIGNATURE	0xaa55
```

[https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/pc/boot.h#L45](https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/pc/boot.h#L45)
```c
/* The offset of the start of the partition table.  */
#define GRUB_BOOT_MACHINE_PART_START	0x1be
```

- It prints a notification message "GRUB ".

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L252](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L252)
```
	/* print a notification message on the screen */
	MSG(notification_string)
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L481](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L481)
```
notification_string:	.asciz "GRUB "
```

- It jumps to 0x8000.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L454](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L454)
```
	/* boot kernel */
	jmp	*(LOCAL(kernel_address))
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L182](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/boot.S#L182)
```
LOCAL(kernel_address):
	.word	GRUB_BOOT_MACHINE_KERNEL_ADDR
```

[https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/pc/boot.h#L63](https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/pc/boot.h#L63)
```c
/* The address where the kernel is loaded.  */
#define GRUB_BOOT_MACHINE_KERNEL_ADDR	(GRUB_BOOT_MACHINE_KERNEL_SEG << 4)
```

[https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/offsets.h#L146](https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/offsets.h#L146)
```c
#define GRUB_BOOT_MACHINE_KERNEL_SEG GRUB_OFFSETS_CONCAT (GRUB_BOOT_, GRUB_MACHINE, _KERNEL_SEG)
```

[https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/offsets.h#L38](https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/offsets.h#L38)
```c
/* The segment where the kernel is loaded.  */
#define GRUB_BOOT_I386_PC_KERNEL_SEG		0x800
```

#### grub-core/boot/i386/pc/diskboot.S
- diskboot.img is located at 0x8000.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L38](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L38)
```
	/* Tell GAS to generate 16-bit instructions so that this code works
	   in real mode. */
	.code16

	.globl	start, _start
start:
_start:
	/*
	 * _start is loaded at 0x8000 and is jumped to with
	 * CS:IP 0:0x8000 in kernel.
	 */
```

- It displays a notification message "loading".

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L38](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L38)
```
	MSG(notification_string)
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L323](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L323)
```
notification_string:	.asciz "loading"
```

- The remaining part of core.img is located in 0x8200 and it jumps to 0x8200.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L365](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L365)
```
	.org 0x200 - GRUB_BOOT_MACHINE_LIST_SIZE
LOCAL(firstlist):	/* this label has to be before the first list entry!!! */
        /* fill the first data listing with the default */
blocklist_default_start:
	/* this is the sector start parameter, in logical sectors from
	   the start of the disk, sector 0 */
	.long 2, 0
blocklist_default_len:
	/* this is the number of sectors to read.  grub-mkimage
	   will fill this up */
	.word 0
blocklist_default_seg:
	/* this is the segment of the starting address to load the data into */
	.word (GRUB_BOOT_MACHINE_KERNEL_SEG + 0x20)
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L64](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/diskboot.S#L64)
```
        /* this is the loop for reading the rest of the kernel in */
LOCAL(bootloop):

	/* check the number of sectors to read */
	cmpw	$0, 8(%di)

	/* if zero, go to the start function */
	je	LOCAL(bootit)
```

#### grub-core/boot/i386/pc/startup_raw.S
- The main part of core.img is located at 0x8200.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L32](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L32)
```
	/* Tell GAS to generate 16-bit instructions so that this code works
	   in real mode. */
	.code16

	.globl	start, _start
start:
_start:
LOCAL (base):
	/*
	 *  Guarantee that "main" is loaded at 0x0:0x8200.
	 */
```

- It changes the CPU mode from 16 bit real mode to 32 bit protected mode.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L97](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L97)
```
	/* transition to protected mode */
	calll	real_to_prot
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/i386/realmode.S#L133](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/i386/realmode.S#L133)
```
real_to_prot:
```

- It enables A20 line.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L104](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L104)
```
	call	grub_gate_a20
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L144](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L144)
```
/*
 * grub_gate_a20(void)
 *
 * Gate address-line 20 for high memory.
 *
 * This routine is probably overconservative in what it does, but so what?
 *
 * It also eats any keystrokes in the keyboard buffer.  :-(
 */

grub_gate_a20:	
```

- It decompresses the GRUB2 kernel into 0x100000.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L332](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/boot/i386/pc/startup_raw.S#L332)
```
post_reed_solomon:

#ifdef ENABLE_LZMA
	movl	$GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR, %edi
#ifdef __APPLE__
	movl	$decompressor_end, %esi
#else
	movl	$LOCAL(decompressor_end), %esi
#endif
	pushl	%edi
	movl	LOCAL (uncompressed_size), %ecx
	leal	(%edi, %ecx), %ebx
	/* Don't remove this push: it's an argument.  */
	push 	%ecx
	call	_LzmaDecodeA
	pop	%ecx
	/* _LzmaDecodeA clears DF, so no need to run cld */
	popl	%esi
#endif

	movl	LOCAL(boot_dev), %edx
	movl	$prot_to_real, %edi
	movl	$real_to_prot, %ecx
	movl	$LOCAL(realidt), %eax
	jmp	*%esi
```

[https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/pc/memory.h#L36](https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/pc/memory.h#L36)
```c
/* The area where GRUB is decompressed at early startup.  */
#define GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR	0x100000
```

#### grub-core/kern/i386/pc/startup.S
[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/i386/pc/startup.S#L55](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/i386/pc/startup.S#L55)
```
	.globl	start, _start, __start
start:
_start:
__start:
```

- It call `grub_main()`.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/i386/pc/startup.S#L124](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/i386/pc/startup.S#L124)
```
	/*
	 *  Call the start of main body of C code.
	 */
	call EXT_C(grub_main)
```

#### grub-core/kern/main.c
- `grub_main()` calls `grub_load_normal_mode()`.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/main.c#L266](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/main.c#L266)
```c
/* The main routine.  */
void __attribute__ ((noreturn))
grub_main (void)
{
/* snipped */
  grub_load_normal_mode ();
  grub_rescue_run ();
}
```

- `grub_load_normal_mode()` invokes `grub_cmd_normal()`.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/main.c#L228](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/kern/main.c#L228)
```c
/* Load the normal mode module and execute the normal mode if possible.  */
static void
grub_load_normal_mode (void)
{
/* snipped */
  grub_command_execute ("normal", 0, 0);
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/command.h#L121](https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/command.h#L121)
```c
static inline grub_err_t
grub_command_execute (const char *name, int argc, char **argv)
{
  grub_command_t cmd;

  cmd = grub_command_find (name);
  return (cmd) ? cmd->func (cmd, argc, argv) : GRUB_ERR_FILE_NOT_FOUND;
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/main.c#L543](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/main.c#L543)
```c
GRUB_MOD_INIT(normal)
{
/* snipped */
  /* Register a command "normal" for the rescue mode.  */
  grub_register_command ("normal", grub_cmd_normal,
			 0, N_("Enter normal mode."));
/* snipped */
}
```

- `grub_cmd_normal() -> grub_enter_normal_mode() -> grub_normal_execute() -> grub_show_menu() -> show_menu() -> grub_menu_execute_entry() -> grub_cmd_boot()`

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/main.c#L315](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/main.c#L315)
```c
/* Enter normal mode from rescue mode.  */
static grub_err_t
grub_cmd_normal (struct grub_command *cmd __attribute__ ((unused)),
		 int argc, char *argv[])
{
/* snipped */
    grub_enter_normal_mode (argv[0]);
/* snipped */
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/main.c#L300](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/main.c#L300)
```c
/* This starts the normal mode.  */
void
grub_enter_normal_mode (const char *config)
{
/* snipped */
  grub_normal_execute (config, 0, 0);
/* snipped */
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/main.c#L261](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/main.c#L261)
```c
/* Read the config file CONFIG and execute the menu interface or
   the command line interface if BATCH is false.  */
void
grub_normal_execute (const char *config, int nested, int batch)
{
/* snipped */
	  grub_show_menu (menu, nested, 0);
/* snipped */
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/menu.c#L887](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/menu.c#L887)
```c
grub_err_t
grub_show_menu (grub_menu_t menu, int nested, int autoboot)
{
/* snipped */
      err1 = show_menu (menu, nested, autoboot);
/* snipped */
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/menu.c#L856](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/menu.c#L856)
```c
static grub_err_t
show_menu (grub_menu_t menu, int nested, int autobooted)
{
/* snipped */
      boot_entry = run_menu (menu, nested, &auto_boot);
/* snipped */
      e = grub_menu_get_entry (menu, boot_entry);
/* snipped */
	grub_menu_execute_entry (e, 0);
/* snipped */
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/menu.c#L205](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/normal/menu.c#L205)
```c
/* Run a menu entry.  */
static void
grub_menu_execute_entry(grub_menu_entry_t entry, int auto_boot)
{
/* snipped */
  if (grub_errno == GRUB_ERR_NONE && grub_loader_is_loaded ())
    /* Implicit execution of boot, only if something is loaded.  */
    grub_command_execute ("boot", 0, 0);
/* snipped */
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/commands/boot.c#L185](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/commands/boot.c#L185)
```c
GRUB_MOD_INIT(boot)
{
  cmd_boot =
    grub_register_command ("boot", grub_cmd_boot,
			   0, N_("Boot an operating system."));
}
```

#### grub-core/commands/boot.c
- `grub_cmd_boot() -> grub_loader-boot() -> (grub_loader_boot_func)()`.
- `grub_loader_boot_func` is set in `grub_cmd_linux()` as described later.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/commands/boot.c#L174](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/commands/boot.c#L174)
```c
/* boot */
static grub_err_t
grub_cmd_boot (struct grub_command *cmd __attribute__ ((unused)),
		    int argc __attribute__ ((unused)),
		    char *argv[] __attribute__ ((unused)))
{
  return grub_loader_boot ();
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/commands/boot.c#L140](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/commands/boot.c#L140)
```c
grub_err_t
grub_loader_boot (void)
{
/* snipped */
  err = (grub_loader_boot_func) ();
/* snipped */
}
```


#### grub-core/loader/i386/linux.c
- "linux" and "initrd" commands are registered.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L1126](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L1126)
```c
static grub_command_t cmd_linux, cmd_initrd;

GRUB_MOD_INIT(linux)
{
  cmd_linux = grub_register_command ("linux", grub_cmd_linux,
				     0, N_("Load Linux."));
  cmd_initrd = grub_register_command ("initrd", grub_cmd_initrd,
				      0, N_("Load initrd."));
  my_mod = mod;
}
```

- "linux" command performs as follows:
	- Load the kernel file.
	- Initialize the relocator which will relocate
	- Set the loader as `grub_linux_boot()`.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L647](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L647)
```c
static grub_err_t
grub_cmd_linux (grub_command_t cmd __attribute__ ((unused)),
		int argc, char *argv[])
{
  grub_file_t file = 0;
  struct linux_i386_kernel_header lh;
/* snipped */
  grub_uint64_t preferred_address = GRUB_LINUX_BZIMAGE_ADDR;
/* snipped */
  file = grub_file_open (argv[0], GRUB_FILE_TYPE_LINUX_KERNEL);
  if (! file)
    goto fail;

  if (grub_file_read (file, &lh, sizeof (lh)) != sizeof (lh))
    {
      if (!grub_errno)
	grub_error (GRUB_ERR_BAD_OS, N_("premature end of file %s"),
		    argv[0]);
      goto fail;
    }
/* snipped */
  if (allocate_pages (prot_size, &align,
		      min_align, relocatable,
		      preferred_address))
    goto fail;
/* snipped */
  len = prot_file_size;
  if (grub_file_read (file, prot_mode_mem, len) != len && !grub_errno)
    grub_error (GRUB_ERR_BAD_OS, N_("premature end of file %s"),
		argv[0]);

  if (grub_errno == GRUB_ERR_NONE)
    {
      grub_loader_set (grub_linux_boot, grub_linux_unload,
		       0 /* set noreturn=0 in order to avoid grub_console_fini() */);
      loaded = 1;
    }
/* snipped */
  return grub_errno;
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/linux.h#L31](https://elixir.bootlin.com/grub/grub-2.06/source/include/grub/i386/linux.h#L31)
```c
#define GRUB_LINUX_BZIMAGE_ADDR		0x100000
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L148](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L148)
```c
/* Allocate pages for the real mode code and the protected mode code
   for linux as well as a memory map buffer.  */
static grub_err_t
allocate_pages (grub_size_t prot_size, grub_size_t *align,
		grub_size_t min_align, int relocatable,
		grub_uint64_t preferred_address)
{
/* snipped */
    grub_relocator_chunk_t ch;
    if (relocatable)
      {
	err = grub_relocator_alloc_chunk_align (relocator, &ch,
						preferred_address,
						preferred_address,
						prot_size, 1,
						GRUB_RELOCATOR_PREFERENCE_LOW,
						1);
	for (; err && *align + 1 > min_align; (*align)--)
	  {
	    grub_errno = GRUB_ERR_NONE;
	    err = grub_relocator_alloc_chunk_align (relocator, &ch, 0x1000000,
						    UP_TO_TOP32 (prot_size),
						    prot_size, 1 << *align,
						    GRUB_RELOCATOR_PREFERENCE_LOW,
						    1);
	  }
	if (err)
	  goto fail;
      }
/* snipped */
    prot_mode_mem = get_virtual_current_address (ch);
    prot_mode_target = get_physical_target_address (ch);
/* snipped */
}
```

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/commands/boot.c#L113](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/commands/boot.c#L113)
```c
void
grub_loader_set (grub_err_t (*boot) (void),
		 grub_err_t (*unload) (void),
		 int flags)
{
/* snipped */
  grub_loader_boot_func = boot;
/* snipped */
}
```

- "initrd" command load the initial RAM disk.

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L1036](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L1036)
```c
static grub_err_t
grub_cmd_initrd (grub_command_t cmd __attribute__ ((unused)),
		 int argc, char *argv[])
{
/* snipped */
  if (grub_initrd_init (argc, argv, &initrd_ctx))
    goto fail;
/* snipped */
  {
    grub_relocator_chunk_t ch;
    err = grub_relocator_alloc_chunk_align (relocator, &ch,
					    addr_min, addr, aligned_size,
					    0x1000,
					    GRUB_RELOCATOR_PREFERENCE_HIGH,
					    1);
    if (err)
      return err;
    initrd_mem = get_virtual_current_address (ch);
    initrd_mem_target = get_physical_target_address (ch);
  }

  if (grub_initrd_load (&initrd_ctx, argv, initrd_mem))
    goto fail;
/* snipped */
  return grub_errno;
}
```

- `grub_linux_boot()` set EIP as `ctx.params->code32_start` (`=0x1000000`).
- The relocator relocates the kernel from 0x100000 to 0x1000000 (`ctx.params->code32-start`).

[https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L402](https://elixir.bootlin.com/grub/grub-2.06/source/grub-core/loader/i386/linux.c#L402)
```c
static grub_err_t
grub_linux_boot (void)
{
/* snipped */
  state.ebp = state.edi = state.ebx = 0;
  state.esi = ctx.real_mode_target;
  state.esp = ctx.real_mode_target;
  state.eip = ctx.params->code32_start;
  return grub_relocator32_boot (relocator, state, 0);
}
```


### Kernel
#### arch/x86/boot/compressed/head_64.S
[https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/boot/compressed/head_64.S#L75](https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/boot/compressed/head_64.S#L75)
```
	.code32
SYM_FUNC_START(startup_32)
/* snipped */
	/*
	 * Setup for the jump to 64bit mode
	 *
	 * When the jump is performed we will be in long mode but
	 * in 32bit compatibility mode with EFER.LME = 1, CS.L = 0, CS.D = 1
	 * (and in turn EFER.LMA = 1).	To jump into 64bit mode we use
	 * the new gdt/idt that has __KERNEL_CS with CS.L = 1.
	 * We place all of the values on our mini stack so lret can
	 * used to perform that far jump.
	 */
	leal	rva(startup_64)(%ebp), %eax
/* snipped */
	pushl	$__KERNEL_CS
	pushl	%eax

	/* Enter paged protected Mode, activating Long Mode */
	movl	$(X86_CR0_PG | X86_CR0_PE), %eax /* Enable Paging and Protected mode */
	movl	%eax, %cr0

	/* Jump from 32bit compatibility mode into 64bit mode. */
	lret
SYM_FUNC_END(startup_32)
```

[https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/boot/compressed/head_64.S#L336](https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/boot/compressed/head_64.S#L336)
```
	.code64
	.org 0x200
SYM_CODE_START(startup_64)
/* snipped */
/*
 * Jump to the relocated address.
 */
	leaq	rva(.Lrelocated)(%rbx), %rax
	jmp	*%rax
SYM_CODE_END(startup_64)
```

[https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/boot/compressed/head_64.S#L550](https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/boot/compressed/head_64.S#L550)
```
	.text
SYM_FUNC_START_LOCAL_NOALIGN(.Lrelocated)
/* snipped */
/*
 * Do the extraction, and jump to the new kernel..
 */
	pushq	%rsi			/* Save the real mode argument */
	movq	%rsi, %rdi		/* real mode address */
	leaq	boot_heap(%rip), %rsi	/* malloc area for uncompression */
	leaq	input_data(%rip), %rdx  /* input_data */
	movl	input_len(%rip), %ecx	/* input_len */
	movq	%rbp, %r8		/* output target address */
	movl	output_len(%rip), %r9d	/* decompressed length, end of relocs */
	call	extract_kernel		/* returns kernel location in %rax */
	popq	%rsi

/*
 * Jump to the decompressed kernel.
 */
	jmp	*%rax
SYM_FUNC_END(.Lrelocated)
```

[https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/boot/compressed/misc.c#L344](https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/boot/compressed/misc.c#L344)
```c
asmlinkage __visible void *extract_kernel(void *rmode, memptr heap,
				  unsigned char *input_data,
				  unsigned long input_len,
				  unsigned char *output,
				  unsigned long output_len)
{
/* snipped */
	return output;
}
```

#### arch/x86/kernel/head_64.S
[https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head_64.S#L44](https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head_64.S#L44)
```
	.text
	__HEAD
	.code64
SYM_CODE_START_NOALIGN(startup_64)
/* snipped */
	/* Now switch to __KERNEL_CS so IRET works reliably */
	pushq	$__KERNEL_CS
	leaq	.Lon_kernel_cs(%rip), %rax
	pushq	%rax
	lretq
```

[https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head_64.S#L78](https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head_64.S#L78)
```
.Lon_kernel_cs:
/* snipped */
.Ljump_to_C_code:
	/*
	 * Jump to run C code and to be on a real kernel address.
	 * Since we are running on identity-mapped space we have to jump
	 * to the full 64bit address, this is only possible as indirect
	 * jump.  In addition we need to ensure %cs is set so we make this
	 * a far return.
	 *
	 * Note: do not change to far jump indirect with 64bit offset.
	 *
	 * AMD does not support far jump indirect with 64bit offset.
	 * AMD64 Architecture Programmer's Manual, Volume 3: states only
	 *	JMP FAR mem16:16 FF /5 Far jump indirect,
	 *		with the target specified by a far pointer in memory.
	 *	JMP FAR mem16:32 FF /5 Far jump indirect,
	 *		with the target specified by a far pointer in memory.
	 *
	 * Intel64 does support 64bit offset.
	 * Software Developer Manual Vol 2: states:
	 *	FF /5 JMP m16:16 Jump far, absolute indirect,
	 *		address given in m16:16
	 *	FF /5 JMP m16:32 Jump far, absolute indirect,
	 *		address given in m16:32.
	 *	REX.W + FF /5 JMP m16:64 Jump far, absolute indirect,
	 *		address given in m16:64.
	 */
	pushq	$.Lafter_lret	# put return address on stack for unwinder
	xorl	%ebp, %ebp	# clear frame pointer
	movq	initial_code(%rip), %rax
	pushq	$__KERNEL_CS	# set correct cs
	pushq	%rax		# target address in negative space
	lretq
```

[https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head_64.S#L356](https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head_64.S#L356)
```c
SYM_DATA(initial_code,	.quad x86_64_start_kernel)
```

#### arch/x86/kernel/head64.c
[https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head64.c#L467](https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head64.c#L467)
```c
asmlinkage __visible void __init x86_64_start_kernel(char * real_mode_data)
{
/* snipped */
	x86_64_start_reservations(real_mode_data);
}
```

[https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head64.c#L530](https://elixir.bootlin.com/linux/v5.17.8/source/arch/x86/kernel/head64.c#L530)
```c
void __init x86_64_start_reservations(char *real_mode_data)
{
/* snipped */
	start_kernel();
}
```

#### init/main.c
[https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L928](https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L928)
```c
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
/* snipped */
	/* Do the rest non-__init'ed, we're now alive */
	arch_call_rest_init();
/* snipped */
}
```

[https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L880](https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L880)
```c
void __init __weak arch_call_rest_init(void)
{
	rest_init();
}
```

- `rest_init()` spawns two threads:
	- init process (pid = 1)
	- kernal thread daemon (called kthreadd) (pid = 2)

```
$ ps aux | head
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  41572  5312 ?        Ss   01:46   0:01 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
root         2  0.0  0.0      0     0 ?        S    01:46   0:00 [kthreadd]
```

[https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L680](https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L680)
```c
noinline void __ref rest_init(void)
{
/* snipped */
	/*
	 * We need to spawn init first so that it obtains pid 1, however
	 * the init task will end up wanting to create kthreads, which, if
	 * we schedule it before we create kthreadd, will OOPS.
	 */
	pid = kernel_thread(kernel_init, NULL, CLONE_FS);
/* snipped */
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
/* snipped */
}
```

[https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L1497](https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L1497)
```c
static int __ref kernel_init(void *unused)
{
/* snipped */
	if (ramdisk_execute_command) {
		ret = run_init_process(ramdisk_execute_command);
		if (!ret)
			return 0;
		pr_err("Failed to execute %s (error %d)\n",
		       ramdisk_execute_command, ret);
	}

	/*
	 * We try each of these until one succeeds.
	 *
	 * The Bourne shell can be used instead of init if we are
	 * trying to recover a really broken machine.
	 */
	if (execute_command) {
		ret = run_init_process(execute_command);
		if (!ret)
			return 0;
		panic("Requested init %s failed (error %d).",
		      execute_command, ret);
	}

	if (CONFIG_DEFAULT_INIT[0] != '\0') {
		ret = run_init_process(CONFIG_DEFAULT_INIT);
		if (ret)
			pr_err("Default init %s failed (error %d)\n",
			       CONFIG_DEFAULT_INIT, ret);
		else
			return 0;
	}

	if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;
/* snipped */
}
```

[https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L163](https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L163)
```c
static char *ramdisk_execute_command = "/init";
```

[https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L1422](https://elixir.bootlin.com/linux/v5.17.8/source/init/main.c#L1422)
```c
static int run_init_process(const char *init_filename)
{
/* snipped */
	return kernel_execve(init_filename, argv_init, envp_init);
}
```


### /sbin/init
```
$ ls -la /sbin/init
lrwxrwxrwx 1 root root 22 May 16 11:14 /sbin/init -> ../lib/systemd/systemd
```



## Links
- [Booting · Linux Inside](https://0xax.gitbooks.io/linux-insides/content/Booting/)
- [LiLeoWang - YouTube](https://www.youtube.com/user/LiLeoWang/videos)
- [Linuxのブートシーケンス - Qiita](https://qiita.com/taichitk/items/b3b69705be0e270e9f6e)
