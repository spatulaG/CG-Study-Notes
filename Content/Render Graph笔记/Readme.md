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

![](https://github.com/spatulaG/CG-Study-Notes/blob/main/Content/Render%20Graph%E7%AC%94%E8%AE%B0/async.png?raw=true)

真正的GPU中，Compute Engine不能使用GPU中的固定功能单元，Graphic Engine的某些阶段（如几何处理、光栅化）遇到瓶颈时，Compute Engine中重叠的任务就可以有效率利用没有使用的GPU资源（如Compute Units、寄存器、带宽），提高GPU的利用率，这个思想源于上一代游戏主机，所以主机游戏通常比同等级PC的性能表现更好。现代图形API也给其他平台带来了类似的功能。

使用Compute Shader的渲染引擎都可以使用Async Compute，随着引擎变得越来越复杂，Compute Shader的使用也越来越多.可以使用Async Compute加速的一个例子是Post-Processing，现在的游戏会大量使用后处理，后处理是在图形渲染管线完成一帧后应用的，经常使用Compute Shader来实现。现在的游戏的另一个常用的技术就是Deferred Rendering。通常在渲染前会有一个Pass使用Compute Shader来计算哪些光源影响了屏幕中的每个像素，这一步也可以使用Async Compute加速。

需要注意让Graphic Queue和Compute Queue利用不同的GPU资源，下图中Shadow maps的瓶颈在光栅化这些固定功能单元，这时执行计算任务是很合适的。

![](https://github.com/spatulaG/CG-Study-Notes/blob/main/Content/Render%20Graph%E7%AC%94%E8%AE%B0/async1.png?raw=true)

下面这种情况效率还可能不如不用Async Compute：

![](https://github.com/spatulaG/CG-Study-Notes/blob/main/Content/Render%20Graph%E7%AC%94%E8%AE%B0/async2.png?raw=true)

让多帧重叠可以进一步提高利用率

![](https://github.com/spatulaG/CG-Study-Notes/blob/main/Content/Render%20Graph%E7%AC%94%E8%AE%B0/async3.png?raw=true)

不过实现时还要考虑目标平台，比如Arm Mali GPU在实现Compute的后处理时就要避免出现Fragment->Compute->Fragment的管线，因为会导致Fragment Idle，一种解决办法是让第N帧的后处理跟第N+1帧的Shadow Map并行，因为Shadow Map瓶颈在光栅化这些固定功能单元，这时执行计算任务是很合适的。

Unity HDRP在Shadow Map时做了Async Compute，包括Build Lighting List、SSR、SSAO、Contact Shadow、Volumetrics Voxelization

有了并行就会有同步，同步的关键就是只在需要的时候进行同步，同步用的不对会严重影响性能。在使用Asyn Compute时，Render Graph可以帮助我们自动调度Async Compute、处理同步点，降低了维护成本，使开发者可以专注图形特性的开发。例如Unity SRP计算同步的大概步骤是：
- 按顺序遍历Render Pass
- 遍历Render Pass读取、写入的资源
- 查找向该资源写入的最后一个Pass，定义为Producer Pass
- 如果Producer Pass在当前Pass之前，同时Producer Pass和当前Pass不在同一个Queue中，就更新同步点

同步点计算完成后，在执行Render Pass之前如果需要同步就会调用WaitOnAsyncGraphicsFence进行同步，在执行Render Pass之后如果有Pass需要同步就会Signal一个Fence。

## Resource Barrier(Memory Barrier)
Barrier用于保证两个job的先后顺序。GPU的两种同步分别是执行同步和资源同步。前面说到的async queue就是执行同步。数据同步需要用api控制，也隐式地保证了执行同步。
数据同步分为三方面：
- 读写顺序
- Cache
- Resource State Transition

读写顺序其实是执行同步，比如一个dispatch写入一个资源然后下一个读取。保证两个的顺序在两者之间插入barrier。

GPU换从光执行同步不够，例如当数据写入后还没有刷新，Cache就不available。Cache内容是latest的时候就是Available，如果在待定管线阶段就已经available那么这个内存就是visible。flush Cache就是让内存avail， invalidate cache就是让它visible。

Resource state可以看作是GPU如何访问一个资源的描述。Resource State Transition指一个资源在不同用途的layout不同。同一个内存不同用处是alias。例如buffer一般是线性的，render target就会一个个block。一些硬件是压缩的格式写入数据，但是只能解压的格式读取。Transition之前必须available，Transition一般发生在L2 Cache。

## Barrier 优化
由于Barrier会导致Cache Flush和State Transition，所以多个Barrier最好合并到一起提交。同步的关键就是只在需要的时候才同步，所以应该尽量减少Barrier，比如Read to Read的Barrier就可以把多个Read状态合并到一起。可以利用隐式的Promotion和Decay，隐式的Promotion和Decay意思是runtime和Driver什么事都不用干，如果一个Common状态的资源，没有等待的Write操作、Cache flush、Layout Change，并且当前的Layout可以被任何Read操作读取，那么这个资源就不需要Transition，Read操作会把资源隐式的Promote到对应的状态。

另一个优化就是把Barrier这堵墙拆成两半，在这两半之间的任务可以并行，也就提高了并行度。

Barrier可以指定srcStageMask和dstStageMask，srcStageMask是需要等待的阶段，dstStageMask是墙的另一半，srcStageMask之前的任务需要在dstStageMask之后的任务开始之前完成。比如在延迟渲染中，在第一个Pass写入G-buffer，在第二个Pass中对它们采样，这时就需要一个Barrier。最简单的方法就是用一个非常保守的Barrier，它会阻塞所有的阶段。这样对性能有很大的影响，因为这会在两个Pass之间使Pipeline Flush。如果第二个Pass只需要在Fragment Shading时采样G-buffer，我们就可以使用一个更宽松的Barrier。
（COLOR_ATTACHMENT_OUTPUT_BIT → FRAGMENT_SHADER_BIT）。这样的Barrier可以让GPU把第一个Pass和第二个Pass的Vertex Shading等阶段并行。这个功能在D3D12中就是各种不同的Read State，比如D3D12_RESOURCE_STATE_NON_PIXEL_SHADER、D3D12_RESOURCE_STATE_PIXEL_SHADER。

拆墙还有另一种方法，Vulkan中是Event，D3D12中叫做Split Barrier。Split Barrier让GPU直到一个处于A状态的资源以后要以B状态来使用，Split Barrier分为Begin和End，Begin Barrier提交后这个资源就不能被访问了，在Begin Barrier和End Barrier之间的其他任务就可以填充Barrier导致的Bubble，提高利用率。Render Graph可以自动管理Resource Barrier并进行优化，最小化Barrier的数量，尽早开始Barrier，把多个Barrier合并提交，这样降低了出错的概率，使代码更清晰，开发者可以专注图形特性的开发。

比如下图的例子，绿线表示Read，红线表示Write，Render Graph可以计算出优化的Barrier，例如Resource A在Pass1中作为Shader Resource使用，在Pass2中作为Render Target使用，那么这两个Pass之间就需要一个Barrier，如果这两个Pass都把Resource A当成Shader Resource使用，那就不需要任何Barrier了。

![v2-432ef1df5e47fba989672506698fdfb4_1440w](https://user-images.githubusercontent.com/29577919/169248572-94e672f8-4d65-4bf4-a4bb-5436ce206904.png)

比如图的例子，绿线表示Read，红线表示Write，Render Graph可以计算出优化的Barrier，例如Resource A在Pass1中作为Shader Resource使用，在Pass2中作为Render Target使用，那么这两个Pass之间就需要一个Barrier，如果这两个Pass都把Resource A当成Shader Resource使用，那就不需要任何Barrier了。


