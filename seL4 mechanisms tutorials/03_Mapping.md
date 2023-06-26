# Mapping

2022.11.28 xzy

## Outcomes

By the end of this tutorial, you should be familiar with:

1. How to map and unmap virtual memory pages in seL4.

## Background

### Virtual memory

seL4 does not provide virtual memory management, beyond kernel primitives for manipulating hardware paging structures. User-level must provide services for creating intermediate paging structures, mapping and unmapping pages.

Users are free to define their own address space layout with one restriction: the seL4 kernel claims the high part of the virtual memory range. On most 32-bit platforms, this is 0xe0000000 and above. This variable is set per platform, and can be found by finding the `kernelBase` variable in the seL4 source.

### Paging structures

As part of the boot process, seL4 initialises the root task with a top-level hardware virtual memory object, which is referred to as a *VSpace*. A capability to this structure is made available in the `seL4_CapInitThreadVSpace` slot in the root tasks CSpace. For each architecture, this capability is to a different object type corresponding to the hardware, top-level paging structure. The table below lists the VSpace object type for each supported architecture.

| Architecture | VSpace Object              |
| :----------- | :------------------------- |
| aarch32      | `seL4_PageDirectory`       |
| aarch64      | `seL4_PageGlobalDirectory` |
| ia32         | `seL4_PageDirectory`       |
| x86_64       | `seL4_PML4`                |
| RISC-V       | `seL4_PageTable`           |

In addition to the top-level paging structure, intermediate hardware virtual memory objects are required to map pages. The table below lists those objects, in order, for each architecture.

| Architecture | Objects                                                      |
| :----------- | :----------------------------------------------------------- |
| aarch32      | `seL4_PageTable`                                             |
| aarch64      | `seL4_PageUpperDirectory`, `seL4_PageDirectory`, `seL4_PageTable` |
| ia32         | `seL4_PageTable`                                             |
| x86_64       | `seL4_PDPT`, `seL4_PageDirectory`, `seL4_PageTable`          |
| RISC-V       | `seL4_PageTable`                                             |

This tutorial covers the x86_64 architecture, but should contain sufficient information on the virtual memory API provided by seL4 to generalise to other architectures.

Each paging structure can be invoked in order to map or unmap it. Below is an example of mapping an x86_64 `seL4_PDPT` object:

```c
    /* map a PDPT at TEST_VADDR */
    error = seL4_X86_PDPT_Map(pdpt, seL4_CapInitThreadVSpace, TEST_VADDR, seL4_X86_Default_VMAttributes);
```

> x86_64系统中的PDPT表（PageDirectoryPointerTable）即第三级页表
>
> [VM学习—实现自己的内核 - 走看看 (zoukankan.com)](http://t.zoukankan.com/dream397-p-14281187.html)
>
> ![image-20221220103105611](03_Mapping.assets/image-20221220103105611-1671526102105-1.png)

All mapping functions take three arguments:

- the VSpace to map the object into,
- the virtual address to map the object at,
- and virtual memory attributes.

If the virtual memory address provided is not aligned to the size of the paging object, seL4 will mask out any unused bits e.g a 4KiB page mapped at 0xDEADBEEF will end up mapped at 0xDEADB000.

Virtual memory attributes determine the caching attributes of the mapping, which is architecture dependent. Alongside the attributes used in this tutorial (`seL4_X86_Default_VMAttributes`) you can find alternative values, in `libsel4`.

### Pages

Once all of the intermediate paging structures have been mapped for a specific virtual address range, physical frames can be mapped into that range by invoking the frame capability. The code snippet below shows an example of mapping a frame at address `TEST_VADDR`.

```c
    /* map a read-only page at TEST_VADDR */
    error = seL4_X86_Page_Map(frame, seL4_CapInitThreadVSpace, TEST_VADDR, seL4_CanRead, seL4_X86_Default_VMAttributes);
```

For a page mapping to succeed, all mid-level paging structures must be mapped. The `libsel4` function `seL4_MappingFailedLookupLevel()` can be used to determine at which level paging structures are missing. Note that to map a frame multiple times, one must make copies of the frame capability: each frame capability can only track one mapping.

In addition to the arguments taken by the map methods for intermediate paging structures, page mapping takes a `rights` argument which determines the mapping type. In the example above, we map the page read only.

#### Types and sizes

Page types and sizes are architecture dependent. For both x86 and ARM architectures, each size of page is a different object type with a specific size. On RISC-V, pages are the same object type and variably sized. Configuration and hardware settings alter the available page sizes.

## Notes

just follow the tutorial.

## Code

```c

#include <stdio.h>
#include <sel4/sel4.h>
#include <sel4platsupport/bootinfo.h>
#include <utils/util.h>
#include <sel4tutorials/alloc.h>
#include <sel4/sel4_arch/mapping.h>
#define TEST_VADDR 0xA000000000

int main(int argc, char *argv[]) {
    /* parse the location of the seL4_BootInfo data structure from
    the environment variables set up by the default crt0.S */
    seL4_BootInfo *info = platsupport_get_bootinfo();
    seL4_Error error;
    seL4_CPtr frame = alloc_object(info, seL4_X86_4K, 0);
    seL4_CPtr pdpt = alloc_object(info, seL4_X86_PDPTObject, 0);
    seL4_CPtr pd = alloc_object(info, seL4_X86_PageDirectoryObject, 0);
    seL4_CPtr pt = alloc_object(info, seL4_X86_PageTableObject, 0);


    /* map a PDPT at TEST_VADDR */
    error = seL4_X86_PDPT_Map(pdpt, seL4_CapInitThreadVSpace, TEST_VADDR, seL4_X86_Default_VMAttributes);

    // DONE  map a page directory object
    error = seL4_X86_PageDirectory_Map(pd, seL4_CapInitThreadVSpace, TEST_VADDR, seL4_X86_Default_VMAttributes);
    assert(error == seL4_NoError);//**solution recommended!

    // DONE  map a page table object
    error = seL4_X86_PageTable_Map(pt, seL4_CapInitThreadVSpace, TEST_VADDR, seL4_X86_Default_VMAttributes);
    assert(error == seL4_NoError);
    /* map a read-only page at TEST_VADDR */
    error = seL4_X86_Page_Map(frame, seL4_CapInitThreadVSpace, TEST_VADDR, seL4_CanRead, seL4_X86_Default_VMAttributes);
    if (error == seL4_FailedLookup) {
        printf("Missing intermediate paging structure at level %lu\n", seL4_MappingFailedLookupLevel());
    }
    ZF_LOGF_IF(error != seL4_NoError, "Failed to map page");

    seL4_Word *x = (seL4_Word *) TEST_VADDR;
    printf("Read x: %lu\n", *x);


    // DONE remap the page
    error = seL4_X86_Page_Map(frame, seL4_CapInitThreadVSpace, TEST_VADDR, seL4_ReadWrite, seL4_X86_Default_VMAttributes);
    assert(error == seL4_NoError);
    
    /* write to the page we mapped */
    printf("Set x to 5\n");
    *x = 5;

    printf("Success!\n");

    return 0;
}

```



## Ref

[API Reference | seL4 docs](https://docs.sel4.systems/projects/sel4/api-doc.html#map-5)

[Mapping | seL4 docs](https://docs.sel4.systems/Tutorials/mapping.html)
