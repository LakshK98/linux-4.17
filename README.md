# Reducing page faults and introducing Cooperative Scheduling in the Linux Kernel

## Reducing page faults with Extent struct
This fork introduces enhancements to the Linux 4.17 kernel, enabling the counting of physically and virtually contiguous pages by using a custom `struct extent` defined in `/include/extent-mod.h`. The extent structure is used in a red-black tree to maintain information about physically contiguous pages allocated by our user-space application. The extent structure is defined below.

``` C
typedef struct extent_node {
	unsigned long start_pfn;
	unsigned long end_pfn;
	struct rb_node node;
} extent_node;
```

The start_pfn and end_pfn denote the starting and ending address of the physical page frame number (PFN) for the pages allocated. The rb_node struct holds the details of a node in a red-black tree. The Linux Kernel implements a red-black tree within the Kernel code with APIs to add, delete, and modify nodes. Red-black trees are reasonably balanced and can grow based on the nodes that are added to them. Hence, they are ideal to maintain a structure like extent. We used these APIs to maintain a tree made of extent struct. 

Additionally, it facilitates adjusting the number of pages allocated per page fault. 
These modifications optimize allocation efficiency by allowing applications to tune the allocation per fault, reducing context-switching costs.

Usage:

Once the modified kernel has been loaded, a custom sys call must be made before an application modifies allocations.
To call the syscall, add the following function definition. 

``` C
#define __NR_identity 333

long identity_syscall(unsigned long count)
{
    return syscall(__NR_identity, count);
}
```

call pass count of allocation per fault to identity_syscall before execution of your application and pass 0 right after execution is complete.

The number of contiguous segments of memory used is tracked by maintaining a red-black tree within the kernel. The count and size of these segments are logged using printk. To view these make sure to run
```
dmesg -n 5
```

## Cooperative Scheduling in the kernel

This introduces a new system called `enable_coop_sched()`  that allows application threads to convey their inactivity to the scheduler. This system call provides a mechanism for a thread to notify the scheduler that it is not actively using CPUs. When the application is deemed inactive, the system call informs the operating system about its reduced priority. This step involves the application signaling the OS to lower its priority, allowing other active applications to be prioritized over it. Internally, this system call reduces the task's weight inside the kernel by using the `reweight_task()` function of the Completely Fair Scheduler. 


