# stivale2 boot protocol specification

The stivale2 boot protocol is an improved version of the stivale protocol which
provides the kernel with most of the features one may need in a *modern*
x86_64 context (although 32-bit x86 is also supported).

## General information

In order to have a stivale2 compliant kernel, one must have a kernel executable,
in the `elf64` or `elf32` format, with a `.stivale2hdr` section (described below), or
a non-ELF kernel providing a valid stivale2 anchor (see below).
Other executable formats are not supported.

stivale2 will recognise whether the ELF file is 32-bit or 64-bit and put the CPU into
the appropriate mode before loading the kernel. In the case of anchored kernels,
the anchor provides information about which CPU mode to use.

stivale2 natively supports (only for 64-bit ELF kernels) and encourages higher half kernels.
The kernel can load itself at `0xffffffff80000000 + phys_base` (as defined in the linker script)
and the bootloader will take care of everything else, no AT linker script directives needed.

`phys_base` represents the actual physical base address of the kernel.
E.g.: having `. = 0xffffffff80000000 + 2M;` will load the kernel at physical address
2MiB, and at virtual address `0xffffffff80200000`.

For relocatable kernels, the bootloader may change the load addresses of the sections
by adding a slide, if the specified location of a segment is not available for use.

If KASLR is enabled, a random slide will be added unconditionally.

If the kernel loads itself in the lower half, the bootloader will not perform the
higher half relocation and protected memory ranges (PMRs) will not be available.

See the protected memory ranges section for more information on those.

## Anchored kernels

An anchored kernel is a non-ELF kernel, of unspecified executable or otherwise format,
which provides an anchor for the bootloader to latch onto in order to gather
information needed to perform kernel loading and handoff.

Anchored kernels are useful for kernels in a.out, flat binary, or otherwise
non-ELF format that support direct loading.

PMRs will not be available for anchored kernels.

The stivale2 anchor can appear anywhere in the executable, but must be aligned on a
16-byte boundary. Only the firstmost anchor that appears in the file is used.
It must be filled in as follows:
```c
struct stivale2_anchor {
    uint8_t anchor[15];         // Must be ASCII sequence "STIVALE2 ANCHOR"
                                // (excluding quotes)
    uint8_t bits;               // What mode to be handed off control in: 32 or 64
    uint64_t phys_load_addr;    // Physical address to load the kernel executable to
    uint64_t phys_bss_start;    // Physical address of beginning of bss section
    uint64_t phys_bss_end;      // Physical address of end of bss section
    uint64_t phys_stivale2hdr;  // Physical address of stivale2 header after kernel is
                                // loaded in memory
};
```

Anchored kernels are otherwise functionally equivalent to ELF counterparts.

## Tag-less mode
In a bootloader-defined way, the kernel may communicate to the bootloader that this
kernel is a _tag-less_ stivale2 kernel. This indicates that the kernel does not contain
a stivale2 header, but it should be assumed these defaults are set:
 - The entrypoint value is set to the ELF entrypoint.
 - The stack is allocated by the bootloader and is 16K, and is mapped at a bootloader-defined address, with the stipulation that the half it is in must be the same as the half the kernel entrypoint is in
 - The flags are set as following:
    1. KASLR is disabled
    2. Higher-half pointers: If the entrypoint's most significat bit of e_entry in the ELF header is set.
    3. Protected memory ranges: Enabled
 - The bootloader should assume the following tags:
    1. The any video header tag, asking for a graphical framebuffer.
    2. The terminal header tag.
    3. The unmap null header tag.
    4. The SMP header tag, without the x2apic flag and with the SMP stack allocation flag.
};

## Kernel entry machine state

### x86_64

`rip` will be the entry point as defined in the ELF file, unless the `entry_point`
field in the stivale2 header is set to a non-0 value, in which case, it is set to
the value of `entry_point`. For anchored kernels, the `entry_point` field of the
header is always utilised.

At entry, the bootloader will have setup paging mappings as such:

```
 Base Physical Address -                    Size                    ->  Virtual address
  0x0000000000000000   - 4 GiB plus any additional memory map entry -> 0x0000000000000000
  0x0000000000000000   - 4 GiB plus any additional memory map entry -> 0xffff800000000000 (4-level paging only)
  0x0000000000000000   - 4 GiB plus any additional memory map entry -> 0xff00000000000000 (5-level paging only)
  0x0000000000000000   -                 0x80000000                 -> 0xffffffff80000000 (No PMRs only)
```

All the mappings are supervisor, read, write, execute (-rwx).

If PMRs are requested, the bottom-most of the above mappings (the one marked as
`(No PMRs only)`) will not be created, and in its place, only mappings
that describe the kernel's loaded ELF segments with appropriately set MMU permissions
will be created. For relocatable kernels, slides will be appropriately taken into
account. PMRs are not available to lower half kernels.

If the "unmap NULL" header tag is specified, then page 0, or the first 4KiB of the
virtual address space, is going to be unmapped.

The bootloader page tables are in bootloader-reclaimable memory, their specific layout
is undefined as long as they provide the above memory mappings.

If the kernel is a position independent executable, the bootloader is free to
relocate it as it sees fit, potentially performing KASLR (as specified by the config).

At entry all segment registers are loaded as 64 bit code/data segments, limits and
bases are ignored since this is 64-bit mode.

The GDT register is loaded to point to a GDT, in bootloader-reserved memory,
with at least the following entries, starting at offset 0:

  - Null descriptor
  - 16-bit code descriptor. Base = `0`, limit = `0xffff`. Readable.
  - 16-bit data descriptor. Base = `0`, limit = `0xffff`. Writable.
  - 32-bit code descriptor. Base = `0`, limit = `0xffffffff`. Readable.
  - 32-bit data descriptor. Base = `0`, limit = `0xffffffff`. Writable.
  - 64-bit code descriptor. Base and limit irrelevant. Readable.
  - 64-bit data descriptor. Base and limit irrelevant. Writable.

The IDT is in an undefined state. Kernel must load its own.

IF flag, VM flag, and direction flag are cleared on entry. Other flags undefined.

PG is enabled (`cr0`), PE is enabled (`cr0`), PAE is enabled (`cr4`),
LME is enabled (`EFER`).
If the stivale2 header tag for 5-level paging is present, then, if available,
5-level paging is enabled (LA57 bit in `cr4`).
If PMRs are requested, then, the NX bit will be enabled (NX bit in `EFER`).

The A20 gate is opened.

PIC/APIC IRQs are all masked.

`rsp` is set to the requested stack as per stivale2 header. If the requested value is
non-null, an invalid return address of 0 is pushed to the stack before jumping
to the kernel.

`rdi` will contain the address of the stivale2 structure (described below).

All other general purpose registers are set to 0.

### IA-32

`eip` will be the entry point as defined in the ELF file, unless the `entry_point`
field in the stivale2 header is set to a non-0 value, in which case, it is set to
the value of `entry_point`. For anchored kernels, the `entry_point` field of the
header is always utilised.

At entry all segment registers are loaded as 32 bit code/data segments.
All segment bases are `0x00000000` and all limits are `0xffffffff`.

The GDT register is loaded to point to a GDT, in bootloader-reserved memory,
with at least the following entries, starting at offset 0:

  - Null descriptor
  - 16-bit code descriptor. Base = `0`, limit = `0xffff`. Readable.
  - 16-bit data descriptor. Base = `0`, limit = `0xffff`. Writable.
  - 32-bit code descriptor. Base = `0`, limit = `0xffffffff`. Readable.
  - 32-bit data descriptor. Base = `0`, limit = `0xffffffff`. Writable.
  - 64-bit code descriptor. Base and limit irrelevant. Readable.
  - 64-bit data descriptor. Base and limit irrelevant. Writable.

The IDT is in an undefined state. Kernel must load its own.

IF flag, VM flag, and direction flag are cleared on entry. Other flags undefined.

PE is enabled (`cr0`).

The A20 gate is enabled.

PIC/APIC IRQs are all masked.

`esp` is set to the requested stack as per stivale2 header. An invalid return address
of 0 is pushed to the stack before jumping to the kernel.

The address of the stivale2 structure (described below) is pushed onto this stack
before the entry point is called.

All other general purpose registers are set to 0.

### aarch64

`IP` will be the entry point as defined in the ELF file, unless the `entry_point`
field in the stivale2 header is set to a non-0 value, in which case, it is set to
the value of `entry_point`.

At entry, the bootloader will have setup paging mappings as such:

```
 Base Physical Address -                    Size                    ->  Virtual address
  0x0000000000000000   - 4 GiB plus any additional memory map entry -> 0x0000000000000000
  0x0000000000000000   - 4 GiB plus any additional memory map entry -> VMAP_HIGH
```
Where `VMAP_HIGH` is specified in the mandatory `stivale2_struct_vmap` tag.

If the kernel is dynamic and not statically linked, the bootloader will relocate it,
potentially performing KASLR (as specified by the config).

The kernel should NOT modify the bootloader page tables, and it should only use them
to bootstrap its own virtual memory manager and its own page tables.

The kernel is entered at EL1.

`VBAR_EL1` is in an undefined state. Kernel must load its own.

`DAIF.{D,A,I,F}` are set/masked.

`SCTLR_EL1.{M,C,I}` are set. Other bits are undefined.
`MAIR` has at least the following entries, in some order:
  * `0b00001100`, used for mmio regions
  * `0b11111111`, used for normal memory

`SP` is set to the requested stack as per stivale2 header.
The `LR` register has an invalid return address.

`SPSel.SP` is 0.

Neither floating point, SIMD nor timer accesses trap to a higher EL than 1.

`X0` will contain the physical address of the stivale2 structure (described below).

All other general purpose registers are undefined.

## Low memory area

For x86_64 and IA-32, stivale2 guarantees that an area of no less than 32 KiB is
free and usable at physical memory address `0x70000`, regardless of what is
specified in the memory map.

This allows the kernel to perform tasks that require Real Mode accessible memory
such as multicore startup.

## Bootloader-reserved memory

In order for stivale2 to function, it needs to reserve memory areas for either internal
usage (such as page tables, GDT, SMP), or for kernel interfacing (such as returned
structures).

stivale2 ensures that none of these areas are found in any of the sections
marked as "usable" or "kernel/modules" in the memory map.

The location of these areas may vary and it is implementation specific, but they
are guaranteed to be in "bootloader reclaimable" entries.

Once the OS is done needing the bootloader, memory map areas marked as "bootloader
reclaimable" may be used as usable memory. These areas are guaranteed to be aligned
to the smallest possible page size (4K on x86_64 and IA-32), for both base and length,
and they are guaranteed to not overlap other sections of the memory map.

## stivale2 header (.stivale2hdr)

The kernel executable shall have a section `.stivale2hdr` which will contain, or
an anchor pointing to, the header that the bootloader will parse.

Said header looks like this:
```c
struct stivale2_header {
    uint64_t entry_point;   // If not 0, this address will be jumped to as the
                            // entry point of the kernel.
                            // If set to 0, the ELF entry point will be used
                            // instead.

    uint64_t stack;         // This is the stack address which will be in ESP/RSP
                            // when the kernel is loaded.
                            // It can only be set to NULL for 64-bit kernels. 32-bit
                            // kernels are mandated to provide a vaild stack.
                            // 64-bit and 32-bit valid stacks must be at least 256 bytes
                            // in usable space and must be 16 byte aligned addresses.

    uint64_t flags;         // Bit 0: Formerly used to indicate whether to enable
                            //        KASLR, this flag is now reserved as KASLR
                            //        is enabled in the bootloader configuration
                            //        instead. Presently reserved and unused.
                            // Bit 1: If set to 1, all pointers, except otherwise noted,
                            //        are to be offset to the higher half. That is,
                            //        their value will be their physical address plus
                            //        0xffff800000000000 with 4-level paging or
                            //        0xff00000000000000 with 5-level paging on x86_64.
                            //        Success for this feature can be tested by checking
                            //        whether the stivale2 struct pointer argument passed
                            //        to the entry point function is in the higher
                            //        half or not.
                            // Bit 2: If set to 1, enables protected memory ranges.
                            //        See the relevant section.
                            // All other bits are undefined and must be 0.

    uint64_t tags;          // Pointer to the first tag of the linked list of
                            // header tags.
                            // See "stivale2 header tags" section.
                            // The pointer can be either physical or, for higher
                            // half kernels, the virtual in-kernel address.
                            // NULL = no tags.
};
```

### Protected memory ranges

Enabling protected memory ranges (bit 2 in the `flags` field of the main header
struct) tells the stivale2-compliant bootloader that the kernel wants the top-most
2 GiB of the address space to be mapped as per its ELF segments, rather than as a
monolithic, all-permissions, block.

Only 64-bit, higher half, ELF (non anchored) kernels can take advantage of this
feature. Requesting this feature for 32-bit, lower half, or non-ELF kernels has
undefined behaviour.

For PMRs on the x86_64 platform, non-readable ranges are not possible, therefore
they are ignored and forced readable in the MMU, but they are still reported back to
the kernel in the struct tag.

### stivale2 header tags

The stivale2 header uses a mechanism to avoid having protocol versioning, but
rather, feature-specific support detection.

The kernel executable provides the bootloader with a linked list of structures,
the first of which is pointed to by the `tags` entry of the stivale2 header.

The bootloader is free to ignore kernel tags that it does not recognise.
The kernel should make sure that the bootloader has properly interpreted the
provided tags, either by checking returned tags or by other means.

Each tag shall begine with these 2 members:
```c
struct stivale2_hdr_tag {
    uint64_t identifier;
    uint64_t next;
};
```

The `identifier` field identifies what feature the tag is requesting from the
bootloader.

The `next` field points to another tag in the linked list. A NULL value determines
the end of the linked list. Just like the `tags` pointer in the stivale2 header,
this value can either be a physical address or, for higher half kernels, the virtual
in-kernel address.

Tag structures can have more than just these 2 members, but these 2 members MUST
appear at the beginning of any given tag.

Tags can have no extra members and just serve as "flags" to enable some behaviour
that does not require extra parameters.

#### Any video header tag

This tag tells the stivale2-compliant bootloader that the kernel has no requirement
for a framebuffer to be initialised.

Depending on the `preference` field, the kernel can either tell the bootloader that
it prefers a graphical linear framebuffer, but the lack of one (or CGA text mode) is also
acceptable (value `0`); or that it prefers the lack of one (or CGA text mode), but
if such is unavailable, a graphical linear framebuffer is also acceptable (value `1`).

Omitting both the any video header tag and the framebuffer header tag means "force
CGA text mode" (where available), and the bootloader will refuse to boot the kernel if
it fails to fulfill that request.

To detect which mode the bootloader ends up picking, either the framebuffer structure tag
or the text mode structure tag (only one of them, see below) is returned to the
kernel.

```c
struct stivale2_header_tag_any_video {
    uint64_t identifier;          // Identifier: 0xc75c9fa92a44c4db
    uint64_t next;
    uint64_t preference;          // 0: prefer linear framebuffer
                                  // 1: prefer no linear framebuffer
                                  //    (CGA text mode if available)
                                  // All other values undefined.
};
```

#### Framebuffer header tag

This tag can be used alongside the "any video" header tag to specify further
framebuffer preferences.

If used alone, without "any video" header tag, this tag mandates a graphical linear
framebuffer and the bootloader will refuse to boot the kernel if it fails to
initialise a framebuffer.

The width, height, and bpp fields are just hints for the preferred resolution, but
the bootloader reserves the right to override these values if no such video mode is
available.

```c
struct stivale2_header_tag_framebuffer {
    uint64_t identifier;          // Identifier: 0x3ecc1bc43d0f7971
    uint64_t next;
    uint16_t framebuffer_width;   // If all values are set to 0
    uint16_t framebuffer_height;  // then the bootloader will pick the best possible
    uint16_t framebuffer_bpp;     // video mode automatically.
    uint16_t unused;
};
```

#### Framebuffer MTRR write-combining header tag

*Note: This tag is deprecated and considered legacy. Use is discouraged and*
*it may not be supported on newer bootloaders.*

The presence of this tag tells the bootloader to, in case a framebuffer was
requested, make that framebuffer's caching type write-combining using x86's
MTRR model specific registers. This caching type helps speed up framebuffer writes
on real hardware.

It is recommended to use this tag in conjunction with the SMP tag in order to
let the bootloader make the MTRR settings uniform across all CPUs.

If no framebuffer was requested, this tag has no effect.

Identifier: `0x4c7bb07731282e00`

This tag does not have extra members.

#### Terminal header tag

If this tag is present the bootloader is instructed to set up a terminal for
use by the kernel at runtime. See "Terminal struct tag" below.

The terminal can run in either framebuffer or text mode, depending on the target
video mode requested.

```c
struct stivale2_header_tag_terminal {
    uint64_t identifier;          // Identifier: 0xa85d499b1823be72
    uint64_t next;
    uint64_t flags;               // Flags:
                                  // Bit 0: set if callback function is provided.
                                  // All other bits are undefined and must be 0.
    uint64_t callback;            // Address of the terminal callback function,
                                  // (see "Terminal struct tag")
};
```

#### 5-level paging header tag

The presence of this tag enables support for 5-level paging, if available.

Identifier: `0x932f477032007e8f`

This tag does not have extra members.

#### Unmap NULL header tag

The presence of this tag tells the bootloader to unmap the first page of the
virtual address space before passing control to the kernel, for architectures
that support paging.

Identifier: `0x92919432b16fe7e7`

This tag does not have extra members.

#### SMP header tag

The presence of this tag enables support for booting up application processors.

```c
struct stivale2_header_tag_smp {
    uint64_t identifier;          // Identifier: 0x1ab015085f3273df
    uint64_t next;
    uint64_t flags;               // Flags:
                                  //   bit 0: 0 = use xAPIC, 1 = use x2APIC (if available)
                                  //   bit 1: if set, allocates a stack of 16K for all cores automatically
                                  // All other bits are undefined and must be 0.
};
```

## stivale2 structure

The stivale2 structure returned by the bootloader looks like this:
```c
struct stivale2_struct {
    char bootloader_brand[64];    // 0-terminated ASCII bootloader brand string
    char bootloader_version[64];  // 0-terminated ASCII bootloader version string

    uint64_t tags;          // Address of the first of the linked list of tags.
                            // see "stivale2 structure tags" section.
                            // NULL = no tags.
};
```

### stivale2 structure tags

These tags work *very* similarly to the header tags, with the main difference being
that these tags are returned to the kernel by the bootloader.

See "stivale2 header tags".

The kernel is responsible for parsing the tags and the identifiers, and interpreting
the tags that it supports, while handling in a graceful manner the tags it does not
recognise.

#### PMRs structure tag

This tag reports to the kernel that the bootloader recognised the PMR flag
in the main header and it has successfully mapped the kernel as per ELF segments.

It additionally reports back the array of ranges and their permissions as it was
mapped by the bootloader. Ranges bases and sizes are at least 4KiB aligned.

```c
struct stivale2_struct_tag_pmrs {
    uint64_t identifier;          // Identifier: 0x5df266a64047b6bd
    uint64_t next;
    uint64_t entries;             // Count of PMRs in following array
    struct stivale2_pmr pmrs[];   // Array of PMR structs
};
```

The PMR struct looks as follows:

```c
struct stivale2_pmr {
    uint64_t base;
    uint64_t length;
    uint64_t permissions;
};
```

The `permissions` field can have one or more of the following bits set, to determine
the range's permissions:

```c
#define STIVALE2_PMR_EXECUTABLE ((uint64_t)1 << 0)
#define STIVALE2_PMR_WRITABLE   ((uint64_t)1 << 1)
#define STIVALE2_PMR_READABLE   ((uint64_t)1 << 2)
```

#### Command line structure tag

This tag reports to the kernel the command line string that was passed to it by
the bootloader.

```c
struct stivale2_struct_tag_cmdline {
    uint64_t identifier;          // Identifier: 0xe5e76a1b4597a781
    uint64_t next;
    uint64_t cmdline;             // Pointer to a null-terminated cmdline
};
```

#### Memory map structure tag

This tag reports to the kernel the memory map built by the bootloader.

```c
struct stivale2_struct_tag_memmap {
    uint64_t identifier;          // Identifier: 0x2187f79e8612de07
    uint64_t next;
    uint64_t entries;             // Count of memory map entries
    struct stivale2_mmap_entry memmap[];  // Array of memory map entries
};
```

###### Memory map entry

```c
struct stivale2_mmap_entry {
    uint64_t base;      // Physical address of base of the memory section
    uint64_t length;    // Length of the section
    uint32_t type;      // Type (described below)
    uint32_t unused;
};
```

`type` is an enumeration that can have the following values:

```c
enum stivale2_mmap_type : uint32_t {
    USABLE                 = 1,
    RESERVED               = 2,
    ACPI_RECLAIMABLE       = 3,
    ACPI_NVS               = 4,
    BAD_MEMORY             = 5,
    BOOTLOADER_RECLAIMABLE = 0x1000,
    KERNEL_AND_MODULES     = 0x1001,
    FRAMEBUFFER            = 0x1002
};
```

All other values are undefined.

The kernel and modules loaded **are not** marked as usable memory. They are marked
as Kernel/Modules (type 0x1001).

The entries are guaranteed to be sorted by base address, lowest to highest.

Usable and bootloader reclaimable entries are guaranteed to be 4096 byte aligned for both base and length.

Usable and bootloader reclaimable entries are guaranteed not to overlap with any other entry.

To the contrary, all non-usable entries (including kernel/modules) are not guaranteed any alignment, nor
is it guaranteed that they do not overlap other entries.

#### Framebuffer structure tag

This tag reports to the kernel the currently set up framebuffer details, if any.

```c
struct stivale2_struct_tag_framebuffer {
    uint64_t identifier;          // Identifier: 0x506461d2950408fa
    uint64_t next;
    uint64_t framebuffer_addr;    // Address of the framebuffer
    uint16_t framebuffer_width;   // Width and height in pixels
    uint16_t framebuffer_height;
    uint16_t framebuffer_pitch;   // Pitch in bytes
    uint16_t framebuffer_bpp;     // Bits per pixel
    uint8_t  memory_model;        // Memory model: 1=RGB, all other values undefined
    uint8_t  red_mask_size;       // RGB mask sizes and left shifts
    uint8_t  red_mask_shift;
    uint8_t  green_mask_size;
    uint8_t  green_mask_shift;
    uint8_t  blue_mask_size;
    uint8_t  blue_mask_shift;
    uint8_t  unused;
};
```

#### Text mode structure tag

This tag reports to the kernel the currently set up CGA text mode details, if any.

```c
struct stivale2_struct_tag_textmode {
    uint64_t identifier;          // Identifier: 0x38d74c23e0dca893
    uint64_t next;
    uint64_t address;             // Address of the text mode buffer
    uint16_t unused;              // Unused, must be 0
    uint16_t rows;                // How many rows
    uint16_t cols;                // How many columns
    uint16_t bytes_per_char;      // How many bytes make up a character
};
```

#### EDID information structure tag

This tag provides the kernel with EDID information as acquired by the firmware.

```c
struct stivale2_struct_tag_edid {
    uint64_t identifier;        // Identifier: 0x968609d7af96b845
    uint64_t next;
    uint64_t edid_size;         // The amount of bytes that make up the
                                // edid_information[] array
    uint8_t  edid_information[];
};
```

#### Framebuffer MTRR write-combining structure tag

*Note: This tag is deprecated and considered legacy. Use is discouraged and*
*it may not be supported on newer bootloaders.*

This tag exists if MTRR write-combining for the framebuffer was requested and
successfully enabled.

Identifier: `0x6bc1a78ebe871172`

This tag does not have extra members.

#### Terminal structure tag

If a terminal was requested (see "Terminal header tag" above), and the feature
is supported, this tag is returned to the kernel to provide it with the physical
entry point of the `stivale2_term_write()` function.

```c
struct stivale2_struct_tag_terminal {
    uint64_t identifier;        // Identifier: 0xc2b3f4c3233b0974
    uint64_t next;
    uint32_t flags;             // Bit 0: cols and rows provided.
                                // Bit 1: max_length provided. If this bit is 0,
                                //        assume a max_length of 1024.
                                // Bit 2: if callback was requested and the
                                //        bootloader supports it, this bit is set.
                                // Bit 3: context control available.
                                // All other bits undefined and set to 0.
    uint16_t cols;              // Columns of characters of the terminal.
                                // Valid only if bit 0 of flags is set.
    uint16_t rows;              // Rows of characters of the terminal.
                                // Valid only if bit 0 of flags is set.
    uint64_t term_write;        // Physical pointer to the entry point of the
                                // stivale2_term_write() function.
    uint64_t max_length;        // If bit 1 of flags is set, this field exists
                                // and it specifies what the maximum allowed
                                // string length that can be passed to term_write() is.
                                // If max_length is 0, then there is no limit.
};
```

The C prototype of this function is the following:

```c
void stivale2_term_write(uint64_t ptr, uint64_t length);
```

The calling convention matches the SysV C ABI for the specific architecture.

The function is not thread-safe, nor reentrant.

##### Terminal callback

The callback is a function that is part of the kernel, which is called by the
terminal during a `term_write()` call whenever an event or escape sequence cannot be
handled by the bootloader's terminal alone, and the kernel may want to be notified in
order to handle it itself.

Returning from the callback will resume the `term_write()` call which will return
to its caller normally.

Not returning from a callback may leave the terminal in an undefined state and cause
trouble.

The callback function must have the following prototype:
```c
void stivale2_terminal_callback(uint64_t type, uint64_t, uint64_t, uint64_t);
```
The purpose of the last 3 arguments changes depending on the first, `type`, argument.

The calling convention matches the SysV C ABI for the specific architecture.

The types are as follows:

* `STIVALE2_TERM_CB_DEC` - (type value: `10`)

This callback is triggered whenever a DEC Private Mode (DECSET/DECRST) sequence
is encountered that the terminal cannot handle alone. The arguments to this
callback are: `type`, `values_count`, `values`, `final`.

`values_count` is a count of how many values are in the array pointed to by `values`.
`values` is a pointer to an array of `uint32_t` values, which are the values passed
to the DEC private escape.
`final` is the final character in the DEC private escape sequence (typically `l` or
`h`).

* `STIVALE2_TERM_CB_BELL` - (type value: `20`)

This callback is triggered whenever a bell event is determined to be necessary
(such as when a bell character `\a` is encountered). The arguments to this
callback are: `type`, `unused1`, `unused2`, `unused3`.

* `STIVALE2_TERM_CB_PRIVATE_ID` - (type value: `30`)

This callback is triggered whenever the kernel has to respond to a DEC private
identification request. The arguments to this callback are: `type`, `unused1`,
`unused2`, `unused3`.

* `STIVALE2_TERM_CB_STATUS_REPORT` - (type value `40`)

This callback is triggered whenever the kernel has to respond to a ECMA-48
status report request. The arguments to this callback are: `type`, `unused1`,
`unused2`, `unused3`.

* `STIVALE2_TERM_CB_POS_REPORT` - (type value `50`)

This callback is triggered whenever the kernel has to respond to a ECMA-48
cursor position report request. The arguments to this callback are: `type`, `x`,
`y`, `unused3`. Where `x` and `y` represent the cursor position at the time the
callback is triggered.

* `STIVALE2_TERM_CB_KBD_LEDS` - (type value `60`)

This callback is triggered whenever the kernel has to respond to a keyboard
LED state change request. The arguments to this callback are: `type`, `led_state`,
`unused2`, `unused3`. `led_state` can have one of the following values:
`0, 1, 2, or 3`. These values mean: clear all LEDs, set scroll lock, set num lock,
and set caps lock LED, respectively.

* `STIVALE2_TERM_CB_MODE` - (type value: `70`)

This callback is triggered whenever an ECMA-48 Mode Switch sequence
is encountered that the terminal cannot handle alone. The arguments to this
callback are: `type`, `values_count`, `values`, `final`.

`values_count` is a count of how many values are in the array pointed to by `values`.
`values` is a pointer to an array of `uint32_t` values, which are the values passed
to the mode switch escape.
`final` is the final character in the mode switch escape sequence (typically `l` or
`h`).

* `STIVALE2_TERM_CB_LINUX` - (type value `80`)

This callback is triggered whenever a private Linux escape sequence
is encountered that the terminal cannot handle alone. The arguments to this
callback are: `type`, `values_count`, `values`, `unused3`.

`values_count` is a count of how many values are in the array pointed to by `values`.
`values` is a pointer to an array of `uint32_t` values, which are the values passed
to the Linux private escape.

##### Terminal context control

If context control is available, the `term_write` function can additionally be
used to set and restore terminal context, and refresh the terminal fully.

In order to achieve these, special values for `length` are passed.
These values are:
```c
#define STIVALE2_TERM_CTX_SIZE ((uint64_t)(-1))
#define STIVALE2_TERM_CTX_SAVE ((uint64_t)(-2))
#define STIVALE2_TERM_CTX_RESTORE ((uint64_t)(-3))
#define STIVALE2_TERM_FULL_REFRESH ((uint64_t)(-4))
```

For `CTX_SIZE`, the `ptr` variable has to point to a location to which the terminal
will *write* a single `uint64_t` which contains the size of the terminal context.

For `CTX_SAVE` and `CTX_RESTORE`, the `ptr` variable has to point to a location
to which the terminal will *save* or *restore* its context from, respectively.
This location must have a size congruent to the value received from `CTX_SIZE`.

For `FULL_REFRESH`, the `ptr` variable is unused. This routine is to be used after
control of the framebuffer is taken over and the bootloader's terminal has to
*fully* repaint the framebuffer to avoid inconsistencies.

##### x86_64

Additionally, the kernel must ensure, when calling this routine, that:

* Either the GDT provided by the bootloader is still properly loaded, or a custom
GDT is loaded with at least the following descriptors in this specific order:

  - Null descriptor
  - 16-bit code descriptor. Base = `0`, limit = `0xffff`. Readable.
  - 16-bit data descriptor. Base = `0`, limit = `0xffff`. Writable.
  - 32-bit code descriptor. Base = `0`, limit = `0xffffffff`. Readable.
  - 32-bit data descriptor. Base = `0`, limit = `0xffffffff`. Writable.
  - 64-bit code descriptor. Base and limit irrelevant. Readable.
  - 64-bit data descriptor. Base and limit irrelevant. Writable.

* The currently loaded virtual address space is still the one provided at entry
by the bootloader, or a custom virtual address space is loaded which identity maps
the framebuffer memory region and all the bootloader reclaimable memory regions, with
read, write, and execute permissions.

* The routine is called *by its physical address* which should be identity mapped.

* Bootloader-reclaimable memory entries are left untouched until after the kernel
is done utilising bootloader-provided facilities (this terminal being one of them).

Notes regarding segment registers and FPU:

The values of the FS and GS segments are guaranteed preserved across the call.
All other segment registers may have their "hidden" portion overwritten, but
stivale2 guarantees that the "visible" portion is going to be restored to the one
used at the time of call before returning.

No registers other than the segment registers and general purpose registers are
going to be used. Especially, this means that there is no need to save and restore
FPU, SSE, or AVX state when calling the terminal write function.

##### IA-32

This service is not provided to IA-32 kernels.

##### Terminal characteristics

It is guaranteed that the terminal be able to print the 7-bit ASCII character set,
that `'\n'` (`0x0a`) is the newline character that puts the cursor to the beginning of
the next line (scrolling when necessary), and that `'\b'` (`0x08`) is the backspace
character that moves the cursor over the previous character on screen.

All other expansions on this basic set of features are implementation specific.

Limine, the reference stivale2 implementation, supports a Linux console compatible
terminal, which is a superset of a VT100 compatible terminal.

#### Modules structure tag

This tag lists modules that the bootloader loaded alongside the kernel, if any.

```c
struct stivale2_struct_tag_modules {
    uint64_t identifier;          // Identifier: 0x4b6fe466aade04ce
    uint64_t next;
    uint64_t module_count;        // Count of loaded modules
    struct stivale2_module modules[]; // Array of module descriptors
};
```

```c
struct stivale2_module {
    uint64_t begin;         // Address where the module is loaded
    uint64_t end;           // End address of the module
    char string[128];       // ASCII 0-terminated string passed to the module
                            // as specified in the config file
};
```

#### RSDP structure tag

This tag reports to the kernel the location of the ACPI RSDP structure in memory.

```c
struct stivale2_struct_tag_rsdp {
    uint64_t identifier;        // Identifier: 0x9e1786930a375e78
    uint64_t next;
    uint64_t rsdp;              // Pointer to the ACPI RSDP structure
};
```

#### SMBIOS structure tag

This tag reports to the kernel the location of the SMBIOS entry points in memory.

```c
struct stivale2_struct_tag_smbios {
    uint64_t identifier;        // Identifier: 0x274bd246c62bf7d1
    uint64_t next;
    uint64_t flags;             // Flags for future use. Currently unused and must be 0.
    uint64_t smbios_entry_32;   // 32-bit SMBIOS entry point address. 0 if unavailable.
    uint64_t smbios_entry_64;   // 64-bit SMBIOS entry point address. 0 if unavailable.
};
```

#### Epoch structure tag

This tag reports to the kernel the current UNIX epoch, as per RTC.

```c
struct stivale2_struct_tag_epoch {
    uint64_t identifier;        // Identifier: 0x566a7bed888e1407
    uint64_t next;
    uint64_t epoch;             // UNIX epoch at boot, read from system RTC
};
```

#### Firmware structure tag

This tag reports to the kernel info about the firmware.

```c
struct stivale2_struct_tag_firmware {
    uint64_t identifier;        // Identifier: 0x359d837855e3858c
    uint64_t next;
    uint64_t flags;             // Bit 0: 0 = UEFI, 1 = BIOS
};
```

#### EFI system table structure tag

This tag provides the kernel with a pointer to the EFI system table
if available.

```c
struct stivale2_struct_tag_efi_system_table {
    uint64_t identifier;        // Identifier: 0x4bc5ec15845b558e
    uint64_t next;
    uint64_t system_table;      // Address of the EFI system table
};
```

#### Kernel file structure tag

This tag provides the kernel with a pointer to a copy of the raw executable file
of the kernel that the bootloader loaded.

```c
struct stivale2_struct_tag_kernel_file {
    uint64_t identifier;        // Identifier: 0xe599d90c2975584a
    uint64_t next;
    uint64_t kernel_file;       // Address of the raw kernel file
};
```

#### Kernel file v2 structure tag

This tag provides information about the raw kernel file that was loaded by the
bootloader.

```c
struct stivale2_struct_tag_kernel_file_v2 {
    uint64_t identifier;       // Identifier: 0x37c13018a02c6ea2
    uint64_t next;
    uint64_t kernel_file;      // Address of the raw kernel file
    uint64_t kernel_size;      // Size of the raw kernel file
};
```

#### Kernel slide structure tag

This tag returns the slide that the bootloader applied over the kernel's load
address as a positive offset.

```c
struct stivale2_struct_tag_kernel_slide {
    uint64_t identifier;        // Identifier: 0xee80847d01506c57
    uint64_t next;
    uint64_t kernel_slide;      // Kernel slide
};
```

#### SMP structure tag

This tag reports to the kernel info about a multiprocessor environment.

```c
struct stivale2_struct_tag_smp {
    uint64_t identifier;        // Identifier: 0x34d1d96339647025
    uint64_t next;
    uint64_t flags;             // Flags:
                                //   bit 0: Set if x2APIC was requested and it
                                //          was supported and enabled.
                                //  All other bits are undefined and set to 0.
    uint32_t bsp_lapic_id;      // LAPIC ID of the BSP (bootstrap processor).
    uint32_t unused;            // Reserved for future use.
    uint64_t cpu_count;         // Total number of logical CPUs (including BSP)
    struct stivale2_smp_info smp_info[];  // Array of smp_info structs, one per
                                          // logical processor, including BSP.
};
```

```c
struct stivale2_smp_info {
    uint32_t processor_id;       // ACPI Processor UID as specified by MADT
    uint32_t lapic_id;           // LAPIC ID as specified by MADT
    uint64_t target_stack;       // The stack that will be loaded in ESP/RSP
                                 // once the goto_address field is loaded.
                                 // This MUST point to a valid stack of at least
                                 // 256 bytes in size, and 16-byte aligned.
                                 // target_stack is an unused field for the
                                 // struct describing the BSP.
    uint64_t goto_address;       // This field is polled by the started APs
                                 // until the kernel on another CPU performs an
                                 // atomic write to this field.
                                 // When that happens, bootloader code will
                                 // load up ESP/RSP with the stack value as
                                 // specified in target_stack.
                                 // It will then proceed to load a pointer to
                                 // this very structure into either register
                                 // RDI for 64-bit or on the stack for 32-bit,
                                 // then, goto_address is called (a bogus return
                                 // address is pushed onto the stack) and execution
                                 // is handed off.
                                 // The CPU state will be the same as described
                                 // in "kernel entry machine state", with the exception
                                 // of ESP/RSP and RDI/stack arg being set up as
                                 // above.
                                 // goto_address is an unused field for the
                                 // struct describing the BSP.
    uint64_t extra_argument;     // This field is here for the kernel to use
                                 // for whatever it wants. Writes here should
                                 // be performed before writing to goto_address
                                 // so that the receiving processor can safely
                                 // retrieve the data.
                                 // extra_argument is an unused field for the
                                 // struct describing the BSP.
};
```

#### PXE server info structure tag

This tag reports that the kernel has been booted via PXE, and reports the server ip that it was booted from.

```c
struct stivale2_struct_tag_pxe_server_info {
    struct stivale2_tag tag;     // Identifier: 0x29d1e96239247032
    uint32_t server_ip;          // Server ip in network byte order
};
```

#### MMIO32 UART tag

This tag reports that there is a memory mapped UART port and its address. To write to this port, write the character, zero extended to a 32 bit unsigned integer to the address provided.

```c
struct stivale2_struct_tag_mmio32_uart {
    uint64_t identifier;        // Identifier: 0xb813f9b8dbc78797
    uint64_t next;
    uint64_t addr;              // The address of the UART port
};
```

#### Device tree blob tag

This tag describes a device tree blob for the platform.

```c
struct stivale2_struct_tag_dtb {
    uint64_t identifier;        // Identifier: 0xabb29bd49a2833fa
    uint64_t next;
    uint64_t addr;              // The address of the dtb
    uint64_t size;              // The size of the dtb
};
```

#### High memory mapping

This tag describes the high physical memory location (`VMAP_HIGH`)

```c
struct stivale2_struct_vmap {
    uint64_t identifier;        // Identifier: 0xb0ed257db18cb58f
    uint64_t next;
    uint64_t addr;              // VMAP_HIGH, where the physical memory is mapped in the higher half
};
```
