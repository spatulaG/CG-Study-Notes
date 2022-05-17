# Render Graph
Frostbite, Unreal, Unity都在使用的基于图的调度系统，提供可视化调试工具。收集一帧的渲染任务，验证管线正确性和剔除冗余工作（未使用的资源就不运行）。

Render Graph不会立刻执行渲染命令，而是先记录完整帧命令之后再编译执行。步骤如下：

- Setup：定义使用的Pass和每个Pass的输入、输出资源。每个Pass用输入输出资源链接起来，把资源用Handle来表示。 定义时不分配内存，由RG来负责内存和资源分配释放。Pass用Lambda定义，以此保证代码流线型，最小化修改
- Compilation：剔除没用的Pass，计算资源生命周期，计算Graphic和Compute的同步点
- Execution：插入Resource Barrier，执行剔除之后

相较于古早的“启发式”算法优化调度GPU（合并状态再提交，CPU向GPU提交命令，内存管理）但API本身是立刻执行（与之相对的是Retained Mode），需要记录跟踪复杂状态产生不必要开销。而且"Applications know more than the driver"，这些算法也不适用于所有情况。现代图形API抛弃了这种模式，把底层GPU的管理交给了应用程序，这就使我们能使用渲染管线的信息在上层优化性能。

简单来说就是交给GPU一步步任务不如让它自己学会找任务（

### Render Graph在优化性能方面总结为以下三点：
- Asynchronous Compute：现代图形API提供的多个Command Queue可以实现Async Compute，Render Graph可以自动调度和同步
- Resource Barrier：Render Graph可以基于整帧的信息来尽早开始Barrier、剔除不必要的Barrier、合并Barrier
- Memory Aliasing：现代图形API使我们可以手动管理内存，Render Graph可以高效的复用内存

## Asynchronous Compute

程序优化的一个大方向就是提高并行度，GPU技术不断发展，已经可以用很低的成本和功耗提供很高的性能，但是就算在执行很密集的渲染任务时，GPU的很大一部分也会是Idle状态，所以通过解决这个问题可以提升GPU的性能，Asyn Compute就是一种方法。GPU在很大程度上依赖管线化的方式来使性能达标，管线中的任意一个阶段出现瓶颈都会使其他阶段不能充分利用。这些阶段包括Command Processing、Geometry Processing、Rasterization、Shading、Raster Operation。每个阶段可以大量并行处理，但是每个阶段的吞吐量（Throughput）还依赖于其他阶段。

现代GPU通常包括多个独立的Engine来提供专用的功能:Engine本质上就是Command Processor，它们可以并行处理命令.
- Graphic(3D) Engine:执行Graphic Pipeline，包括VS、光栅化、PS等
- Compute Engine:Compute Pipeline只有一个阶段：Compute Shade
- 一个或多个Copy Engine:拷贝资源

```
每个厂商实现这个功能的方法可能很不一样，AMD的GCN（Graphics Core Next）架构使用一个叫做“Asynchronous Compute Engine”来处理[1]
NVIDIA的Pascal架构用的是“Dynamic Load Balancing”[2]
Arm使用两个Queue，一个Queue处理Vertex、Tiling、Compute，另一个处理Fragment[3]
PC和Mobile的Async Compute有比较大的区别，实现时需要针对目标平台。
图形API提供了Async Compute的功能，运行时GPU也得支持Async Compute才能得到性能的提升。
```
DirectX 12中提供了三种Command Queue驱动这三个Engine:
- 3D（Graphic） Queue可以驱动这三个Engine
- Compute Queue可以驱动Compute和Copy Engine
- Copy Queue只能驱动Copy Engine

向3D Queue提交一个3D Command List和Compute Command List会使3D Queue启动3D Engine和Compute Engine，但不是同一时间

另一种方法是向3D Queue提交一个3D Command List，向Compute Queue提交一个Compute Command List，这样的结果就是3D Queue驱动3D Engine，同时Compute Queue驱动Compute Engine，这种行为就叫做**Async Compute**

Async Compute的目的是让开发者在比线程更高的层次上实现并行，提高GPU的利用率。
