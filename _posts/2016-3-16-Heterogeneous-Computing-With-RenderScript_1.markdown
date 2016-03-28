---

layout: post
title: 移动端密集计算-RenderScript高性能异构计算框架(一)
author: 方赳

--- 

### __异构计算__
为达到更到的运算性能，在移动端多核CPU, GPU和各种DSP等异构系统越来越普遍。为了屏蔽这些本地异构运算单元之间差异，我们将运算密集型任务分割成子任务，并分配到各种可用的硬件上，来达到运算负载均衡的目的。对于本地异构计算的具体技术细节请参考OpenCL相关的内容，这里为了介绍RenderScript只抛砖引玉做一个简单介绍。

### __Android各版本RenderScript的性能指标__

+ 对大约170万像素的位图进行逐像素运算，对比Android4.0，Android4.1和Android4.2，运算能力的提升：

![Image Processing Result](http://gtms04.alicdn.com/tps/i4/TB1nrYWLVXXXXanXFXXlh3rNVXX-400-278.png)

相对Android4.0版本，Android4.1版本减小了从Allocations中读取元素的开销。Android4.2版本增加了一些性能改进的数学库，ARM还针对Renderscript特别提供了大量的编译器改进，这大大提高了矢量生成代码的能力。

+ Android4.2还引入了一项重要的改进，第一次将GPU正式作为Renderscript的计算设备，我们不需要做特别的事情就可以利用GP-GPU强大的计算能力。下面是CPU+GPU对比只用CPU在Nexus10上的性能提升表现：

![GPU_CPU](http://gtms01.alicdn.com/tps/i1/TB1u6rQLVXXXXanXVXXlh3rNVXX-400-278.png)

不过也并不是所有类型的RenderScript脚本都可以在GPU上运行，Renderscript运行时会自动分析这些脚本来决定需不需要GPU的参与。按照经验，不涉及元素间复杂逻辑关系的运算(单元素四则运算、卷积等)都可以运用GPU的运算能力。

### __RenderScript__

+ RenderScript作为Android平台上的高性能计算密集型任务的框架，在日常开发中使用率很低，相关文档介绍也不是太多，导致很多人对它不太了解。其实从Android3.0开始，我们就已经可以利用RenderScript编写运算密集程序，来达到甚至超过Native的运行效率。它采用c99语法，代码在机器上进行第一遍编译，然后在目标设备上进行最后一遍编译(JIT)，最终产生高效原生二进制执行代码。这也意味着用RenderScript编写程序的时候完全不需要关系具体平台的架构(arm/x86/mips)，相对NDK有更好的平台兼容性。

+ 按照Google对RenderScript的解释，RenderScript程序理论上会优于运行在CPU上的native程序，[Google Guide-About RenderScript](http://developer.android.com/guide/topics/renderscript/compute.html#writing-an-rs-kernel")：

> RenderScript运行时会将任务并行地运行在设备可用的多核CPU、GPU、DSP等运算单元上。让开发者从资源分配和运算负载均衡工作上解放出来，更专注于具体算法逻辑的设计，是图像处理、计算摄影(自动全景拼接)、计算机视觉的首选方案。

+ __RenderScript结构和数据类型__
   + __脚本结构__：
      + __预处理描述__：描述脚本的版本号和脚本编译成Java类后的包名
      
            ＃pragma version(1)
            ＃pragma rs java_package_name(mobi.daogu.gaussblurstrict.rsc)
            
      + __定义内核方法__：内核方法用于处理每一个并行的子任务，以一个类型数组作为输入参数。定义完内核方法后，预编译后会生成一个forEach_funcname(Allocation alloc)方法。其中alloc是Java端输入的数组，运行时Renderscript会根据alloc数组中元素的个数创新相应个数的子任务来调用Renderscript端的funcname()方法。

      + __头文件/宏__：RenderScript支持像C一样定义和引用头文件，头文件后缀为"rsh"。也支持像C一样的宏定义。
      
   + __数据类型__：Renderscript采用的是c99语法，变量申明和C语言方式是一样的，也可以像C一样使用指针和结构体。和C语言不同的是，Renderscript还原生支持矢量类型、矩阵类型的数据。
       + __标量__：uint16_t,uint32_t,int16_t,int32_t,float等，通过类型名称很容易找到和C语言对应的类型，它们的使用方法也并没有什么差别。
       + __矢量__：uchar2,uchar3,uchar4,float2,float3,float4等
       + __矩阵__：rs_matrix2x2,rs_matrix3x3,rs_matrix4x4
   + __数据运算__：+-*/运算符号直接可以用于标量、矢量、矩阵类型之间。针对矢量、矩阵之间的运算，Renderscript有很多优化策略。比如两个uchar4类型的变量相加，用C语言实现可实现为两个数组相加，需要多条指令完成数组的每个元素间的相加，而Renderscript通过硬件加速实现时只需要一条矢量运算指令就可以完成，这很大程度上增强了矢量运算能力，在像素通道处理、三维顶点变换上有很大优势。
+ __宿主程序(Java)和RenderScript__
   * RenderScript类型在Java端的定义：

   ![GPU_CPU](http://gtms04.alicdn.com/tps/i4/TB1wdIuLVXXXXX2XXXXv5KCKXXX-572-334.jpg)
   
   * __Element__: Element是Allocation中其中一项元素。如果RenderScript运行在CPU上，那么可以认为它是RenderScript内核中C基本类型和由它们构成的矢量类型(结构体)。
   
   * __Type__: Type用于描述Allocation的尺寸和Element类型。eg.创建一个100x100的元素为float32类型的Allocation类型：
       
         Type.Builder builder = new Type.Builder(rs, Element.F32(rs));
         Type type = builder.setX(100).setY(100).create();

   * __Allocation__: Allocation是主程序和RenderScript内核之间数据块传递的主要载体。RenderScript内核可能会将Allocation数据块绑定到异构处理单元的数据存储器中(内存，显存，DSP数据存储区等)。通过bindAllocation(Allocation va, int slot)将一个Allocation绑定到Renderscript里的一个对象数组上。
   
         ScriptC.bindAllocation(alloc_input, mExportVarIdx_input);
         ScriptC.bindAllocation(alloc_output, mExportVarIdx_output);