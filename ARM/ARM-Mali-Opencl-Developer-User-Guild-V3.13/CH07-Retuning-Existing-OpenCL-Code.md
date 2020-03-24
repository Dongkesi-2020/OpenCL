# Ch07 Retuning Existing OpenCL Code

本章介绍如何重新调整现有的OpenCL代码，以便可以在Mali GPU上运行它。

## 7.1 About retuning existing OpenCL code for Mali GPUs

OpenCL是一种可移植的语言，但并不总是性能可移植的。 这意味着OpenCL应用程序可以在许多不同类型的计算设备上运行，但性能无法保留。

现有的OpenCL通常针对特定的体系结构进行调整，例如台式机GPU。

###  7.1.1 Converting the existing OpenCL code for Mali™ GPUs

为了使用适用于Mali GPU的OpenCL代码获得更好的性能，您必须重新调整代码。
1. 分析代码。
2. 找到并删除替代计算设备的优化。
3. 优化Mali GPU的OpenCL代码。

为了获得最佳性能，请编写针对特定目标设备优化的内核。

:star: 注意：为了在Mali Midgard GPU上获得最佳性能，必须对代码进行向量化处理。您不需要对Mali Bifrost GPU的代码进行向量化处理，但是对代码进行向量化处理不会降低其性能。

## 7.2 Differences between desktop-based architectures and Mali GPUs

基于桌面的GPU和Mali GPU之间存在一些差异。由于存在这些差异，因此对于Mali GPU，必须以不同的方式对OpenCL进行编程。

### 7.2.1 About desktop-based GPU architectures

台式机GPU的电源可用性和大芯片面积使它们具有与移动GPU不同的特性。

台式机GPU具有：
- 较大的芯片面积
- 大量的Shader核心
- 高带宽内存。
  
台式机GPU的大量功率预算使它们具有这些功能。

台式机GPU具有将**线程放入线程组**中的Shader体系结构。这些被称为warps或wavefronts。这种机制意味着线程必须以lock-step运行。如果没有，例如，如果代码中存在分支，并且线程采用不同的分支，则认为线程是发散的(divergent)。

当线程发散时，两个操作将拆分，并且必须计算两次。这使处理速度减半。

台式机GPU上的内存是按层次结构组织的。数据从主存储器加载到本地存储器中。本地存储器按划分的Bank组织，因此线程组中每个线程只有一个Bank。线程可以访问为其他线程保留的Banks，但是在这种情况下，访问将被序列化，从而降低性能。

### 7.2.2 About Mali GPU architectures 

Mali GPU使用SIMD架构。 指令同时对多个数据元素进行操作。

峰值吞吐量取决于Mali GPU类型和配置的硬件实现。

Mali GPU包含1至16个相同的shader核心。 每个shader内核最多支持384个同时执行的线程。
   
每个shader核心包含：
- 一到三个 arithmetic pipeline。
- 一个load-store pipeline。
- 一个texture pipeline。

:star: 注意：OpenCL通常**仅使用算术或负载存储执行Pipeline**。 纹理Pipeline仅用于读取图像数据类型。

Mali GPU使用VLIW（超长指令字）架构。每个指令字包含多个操作。Mali GPU还使用SIMD，因此大多数算术指令可同时对多个数据元素进行操作。

每个线程在任何时间点仅使用一个算术或负载存储执行Pipeline。来自同一线程的两条指令按顺序执行。

### 7.2.3 Programming OpenCL for Mali GPUs 

在Mali GPU和台式机GPU上对OpenCL进行编程之间存在一些差异。

在Mali GPU上：
- 全局和本地OpenCL地址空间映射到相同的物理内存，并由L1和L2缓存加速。这意味着您不需要使用显式数据拷贝或实现关联的屏障同步。
- 所有线程都有单独的程序计数器。这意味着分支发散不是主要问题。 对于基于warp或wavefront体系结构，分支发散是一个主要问题。

  :star: 注意：在OpenCL中，每个工作项通常映射到Mali GPU上的单个线程。

- 使用内核自动向量化。
  
  :star: 注意：此功能不适用于Bifrost GPU。

## 7.3 Procedure for retuning existing OpenCL code for Mali GPUs

如果您分析现有代码并删除特定于设备的优化，则可以为Mali GPU优化现有的OpenCL代码。

### 7.3.1 Analyze code

如果您不是自己编写代码，则必须对其进行分析以查看其功能。

尝试了解以下内容：
- 代码的目的。
- 算法的工作方式。
- 如果没有优化，代码会怎样。

此分析可以作为指南，帮助您删除特定于设备的优化。

这种分析可能很困难，因为高度优化的代码可能非常复杂。

### 7.3.2 Locate and remove device optimizations

对于替代计算设备有一些优化，这些优化对Mali GPU没有影响，或者会降低性能。 要重新调整Mali GPU的OpenCL代码，必须首先删除所有类型的优化来创建非特定于设备的参考实现。

这些优化如下（需要删除）： 
- Use of local or private memory

  Mali GPU使用Cache而不是本地内存。OpenCL本地和私有内存映射到主内存。因此，在Mali GPU的OpenCL代码中使用本地或私有内存**不会带来性能优势**。

  您可以将本地或私有内存用作临时存储，但是与内存之间的内存复制是一项昂贵的操作。**使用本地或私有内存会降低Mali GPU上OpenCL的性能**。
  
  不要将本地或专用内存用作Cache，因为这会降低性能。处理器已经包含执行相同任务的硬件缓存，而没有昂贵的复制操作的开销。
  
  一些代码将数据复制到本地或私有内存中，进行处理，然后再次将其写出。通过执行这些拷贝，这些代码浪费了性能和Power。
- Barriers
  
  与本地或私有存储器之间的数据传输通常与Barrier同步。如果您删除对这些内存的复制操作，也请删除相关的Barrier。

- Cache size optimizations

  一些代码优化了读写操作，以确保数据适合高速缓存行。这对于提高性能和降低功耗都是非常有用的优化。但是，该代码可能针对与Mali GPU使用的缓存行大小不同的缓存行大小进行了优化。
  
  如果代码针对错误的缓存行大小进行了优化，则可能会发生不必要的缓存刷新，这会降低性能。
  
  :star: 注意：Mali GPU的缓存行大小为64字节。

- Use of scalars

  Mali Bifrost GPU使用标量。
  
  Mali Midgard GPU使用标量和128位向量。
- Modifications for memory bank conflicts

  一些GPU包含per-warp memory banks。如果代码包含优化措施以避免这些banks中的冲突，请删除它们。

- Optimizations for divergent threads, warps, or wavefronts

  一些GPU架构将工作项组合在一起，称为warps或wavefronts。

  在这些体系结构中，所有warp中的所有工作项都必须同步进行，这意味着分支可能会表现不佳。
  
  Mali Midgard GPU上的线程是独立的，并且可以发散而不会影响性能。如果您的代码包含针对warp或wavefronts中的发散线程的优化或替代方法，请删除它们。
  
  :star: 注意：Mali Midgard GPU不使用warps或wavefronts。

  :star: 注意：此优化仅适用于Mali Midgard GPU。

### 7.3.3 Optimize your OpenCL code for Mali GPUs

重新调整代码后，性能会提高。 要提高性能，必须对其进行优化。