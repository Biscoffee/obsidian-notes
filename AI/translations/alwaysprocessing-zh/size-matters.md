# Size Matters: An Exploration of Virtual Memory on iOS

> 原文：https://alwaysprocessing.blog/2022/02/20/size-matters
> 发布：2022-02-20　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

**尺寸很重要：iOS虚拟内存探索**

调试iOS应用时的一次内存不足崩溃事件，促使我对iOS虚拟内存系统进行了一番探究，并发现虚拟地址空间的大小在不同iOS设备上存在差异。要修复调试工作流，就需要使用“扩展虚拟寻址权限”（Extended Virtual Addressing entitlement）以启用完整的64位地址空间。

前几天，我尝试在设备上调试一个iOS应用时，遇到了一个奇怪的内存不足问题。应用一启动就很快崩溃，让我无法深入调查这个bug。为了让自己能继续工作，我深入了解了iOS虚拟内存的实现机制，并在此记录下我的发现（包括解决方案！）。

**背景**

“虚拟内存”（virtual memory）这一术语描述了进程（可执行文件、应用等）与计算机物理内存（RAM）之间的一种抽象层。这是操作系统与计算机CPU（具体来说是其内存管理单元，或称MMU）共同提供的一项功能。

在具有虚拟内存的系统中，每个进程拥有自己的内存地址空间，该空间定义了有效的逻辑（原称虚拟）内存地址。当进程从逻辑内存地址读取或写入数据时，该逻辑地址会被 MMU（内存管理单元）转换为物理地址。这正是使用“虚拟”一词的原因——进程的内存地址本质上与机器上的任何物理地址无关。

这一抽象的推论在于：两个进程可能拥有相同值（即相同的逻辑内存地址）的指针，但每个指针可能映射到不同的物理地址。类似地，两个进程可能拥有不同值的指针，但每个指针可映射到同一物理地址（例如共享库的情况）。

虚拟内存被划分为称为页的单位。页是一段连续的地址范围，且每个页的大小相同且固定。页的大小因操作系统和硬件而异。32位硬件上常见4 KiB，而64位硬件上的常见大小包括4 KiB、16 KiB和64 KiB。

并非虚拟内存地址空间中的所有地址都需要映射到物理地址。如果进程访问未映射页中的地址，MMU会引发页错误异常。

操作系统可利用页错误异常实现分页（paging），即数据从磁盘载入主内存的过程（通常称为换入 page in，或从磁盘读取页面到 RAM）。该技术使操作系统和应用程序能使用的内存超过机器物理容量——通过将操作系统希望从 RAM 中淘汰的页面存储到磁盘（通常称为换出 page out，或将页面从 RAM 移至磁盘）。在 RAM 与磁盘间传输数据的过程也被称为交换（swapping）。

操作系统也可能通过终止进程（即发生崩溃）来处理页错误。在 32 位 Apple 平台上，第一个页面（地址范围 [0x0000, 0x0FFF]）不可被进程访问，因此空指针解引用（NULL pointer dereference，即访问地址 0）会导致崩溃。在 64 位 Apple 平台上，整个 4 GiB 的 32 位地址空间（地址范围 [0x00000000, 0xFFFFFFFF]）均不可被进程访问，这能同时捕获空指针解引用错误和 64 位至 32 位的指针截断错误。

**iOS 上的虚拟内存**

**No Page Out（无页面换出）**

与 macOS 及多数支持虚拟内存的操作系统不同，iOS 不会将脏内存（指进程已写入过的内存）页面换出到磁盘上。它仅丢弃内存映射文件中的只读页面，这些页面在需要时可被再次换入。主可执行文件、共享库及某些类型的资源通常通过内存映射方式加载到进程地址空间中，因此当系统物理内存不足时，它们是可能从 RAM 中被回收的候选对象。

**64-bit Virtual Address Space（64 位虚拟地址空间）**

iOS 64 位虚拟内存系统的一个令人惊讶的方面（至少对我来说是！）在于，其虚拟地址空间的大小取决于设备上安装的物理内存量。请参考 xnu-7195.141.2（操作系统内核）中的相关节选：

```
#define SHARED_REGION_BASE_ARM64                0x180000000ULL
#define SHARED_REGION_SIZE_ARM64                0x100000000ULL
```

```
#define ARM64_MIN_MAX_ADDRESS (SHARED_REGION_BASE_ARM64 + SHARED_REGION_SIZE_ARM64 + 0x20000000) // end of shared region + 512MB for various purposes
const vm_map_offset_t min_max_offset = ARM64_MIN_MAX_ADDRESS; // end of shared region + 512MB for various purposes

if (arm64_pmap_max_offset_default) {
	max_offset_ret = arm64_pmap_max_offset_default;
} else if (max_mem > 0xC0000000) {
	max_offset_ret = min_max_offset + 0x138000000; // Max offset is 13.375GB for devices with > 3GB of memory
} else if (max_mem > 0x40000000) {
	max_offset_ret = min_max_offset + 0x38000000;  // Max offset is 9.375GB for devices with > 1GB and <= 3GB of memory
} else {
	max_offset_ret = min_max_offset;
}
```

利用上述信息[1]，我们可以计算出运行 iOS 12 或更高版本的各类 iOS 设备的虚拟内存地址空间大小。（对于 iOS 11 及更早版本，需减去 3 GiB。）

```
 > 3 GiB
```

```
15.375 GiB
```

iPhone XS – iPhone 13

iPad Air（第四代）

iPad Pro（12.9 英寸）、（10.5 英寸）、（11 英寸）

```
 > 1 GiB
```

```
11.375 GiB
```

iPhone 6s – X, SE, XR

iPad（第 5 代）– iPad（第 8 代）

iPad Air 2、iPad Air（第 3 代）

iPad mini 4、iPad mini（第 5 代）

iPad Pro（9.7 英寸）

```
<= 1 GiB
```

```
10.5   GiB
```

iPhone 5s、iPhone 6  
iPad Air  
iPad mini 2、iPad mini 3  

**可用地址空间**  
虚拟地址空间中有 8 GiB[2] 无法供进程使用。如前所述：  

64 位虚拟地址空间的前 4 GiB（这也恰好是整个 32 位地址空间！）进程无法读取、写入或执行。Mach-O 可执行文件格式将此区域指定为 `PAGE_ZERO`，且内核要求 arm64 进程必须保留 `PAGE_ZERO`。  

供系统使用的共享区域（shared region）大小固定为 4 GiB。  

根据这些信息，我们可以更新表格，加入进程可用的虚拟地址空间大小。

```
 > 3 GiB
```

```
15.375 GiB
```

```
7.375 GiB
```

iPhone XS – iPhone 13

iPad Air（第 4 代）

iPad Pro（12.9 英寸）、（10.5 英寸）、（11 英寸）

```
 > 1 GiB
```

```
11.375 GiB
```

```
3.375 GiB
```

iPhone 6s – X, SE, XR

iPad（第5代）– iPad（第8代）

iPad Air 2, iPad Air（第3代）

iPad mini 4, iPad mini（第5代）

iPad Pro（9.7英寸）

```
<= 1 GiB
```

```
10.5   GiB
```

```
2.5   GiB
```

iPhone 5s, iPhone 6

iPad Air

iPad mini 2, iPad mini 3

开发考虑事项

一般来说，当一个Mach-O文件（macOS和iOS可执行文件格式）加载到进程中时，整个文件会被映射到进程的地址空间。这对调试版本（debug builds）尤其重要，因为调试版本通常包含大量嵌入到可执行二进制文件中的调试信息。

最后，我们回到这篇文章的起源！🤣 我试图在iPhone X上调试一个大型应用，但它会在启动后不久崩溃，并显示这个相当隐晦的错误消息：

```
 can't allocate region
 :*** mach_vm_map(size=1048576, flags: 100) failed (error code=3)
 MyApp(1234,0x2d580000) malloc: *** set a breakpoint in malloc_error_break to debug
 warning: could not execute support code to read Objective-C class data in the process. This may reduce the quality of type information available.
```

使用 Instruments 工具进行分析时，我观察到该应用程序的可执行文件及其库的总大小超过 3 GiB，导致堆和线程栈可用的地址空间不足 370 MiB。我无法在设备上调试该应用，因为调试信息占据了过多地址空间，导致分配器实际上没有可用的地址空间！

**扩展虚拟地址**

iOS 14 新增了“扩展虚拟地址”（Extended Virtual Addressing）权限，其对应的 plist 键为 `com.apple.developer.kernel.extended-virtual-addressing`。如果一个进程拥有此权限，内核将启用巨型模式（jumbo mode），从而使该进程能够访问完整的 64 位地址空间。

```
} else if (option == ARM_PMAP_MAX_OFFSET_JUMBO) {
	if (arm64_pmap_max_offset_default) {
		// Allow the boot-arg to override jumbo size
		max_offset_ret = arm64_pmap_max_offset_default;
	} else {
		max_offset_ret = MACH_VM_MAX_ADDRESS;     // Max offset is 64GB for pmaps with special "jumbo" blessing
	}
```

因为这个权限（entitlement）仅增加虚拟地址空间（virtual address space）的大小，它主要适用于需要映射大量只读数据（read-only data）的应用程序。

我的大型调试版本（debug build）恰好在映射大量只读数据，因此我将此权限添加到了开发版本的应用权限文件中。该权限解决了在设备上调试时启动后发生的内存不足崩溃问题！🚀 🎉 🎊 搞定这个问题后，我得以继续研究那个让我走上这段迷人岔路的 bug。