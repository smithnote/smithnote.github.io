---
layout: archive
title: CUDA基础
date: 2021-10-17 20:15
categories: programing
tag: tools, library
---

在使用[TensorRT](https://developer.nvidia.com/zh-cn/tensorrt)推理库的过程中，我们使用了涉及CUDA相关的接口知识，所有有必要学习和了解CUDA的基本编程原理和知识，特此整理

##  CPU和GPU

* CPU：CPU存在大量的控制和储存单元，导致其拥有强大的分支处理能力，通常用来处理复杂的逻辑分支运算，核心少，但通用性强
* GPU：GPU存在少量的控制和储存单元，但其拥有众多的计算单元，能够并行的处理大规模的简单运算的任务，核心多，擅长并行计算
* 协同工作：复杂的逻辑处理交由CPU处理，简单但庞大的计算处理交由GPU处理，它们之间通过PCI总线交换数据

## GPU硬件

CUDA编程程序是运行于GPU上的，所以在了解CUDA之前，需要了解一些GPU的硬件的基本知识

* GPU中属于多线程流式多处理器的硬件（stream multiprocess, SM）
* GPU卡有多个SM，一个SM(流式多处理器)中有几十甚至上百个SP（single processor），在程序运行过程中，一个SP执行一个CUDA线程，而一张GPU可能有上百甚至上千的SP，也就是能够同时执行上百甚至上千个CUDA线程
* GPU采用的计算架构是SIMT，SIMD的升级版本
  * SIMD，单指令多数据，利用CPU的寄存器一次计算循环去多个数据，一起执行相同的操作
  * SIMT，单指令多线程，同时开启多个线程，每个线程都执行相同的指令，因为SP多，所以适合该架构

## CUDA编程

CUDA编程语言可以视作为CPP语言的拓展，其语法兼容大部分的cpp语法（和编译器版本相关），CUDA源码使用NVCC编译器进行编译运行，在CUDA编程中有以下一些基本概念

* HOST端：指CPU端

* DEVICE端：指GPU端

* KERNEL函数：运行在GPU上的函数，其天然的是多线程直接执行的，具体线程结构在调用的时候指定
  * kernel函数定义或声明的时候，在函数前加`__device__ `或者 `__global__`关键字
  * 调用的时候指定线程结构，其在函数名后加<<<...>>>信息
  * 每个kernel函数都内置自带两个变量blockIdx, threadIdx，其具体值在运行在GPU上时，每个线程的这两个变量值是不一样


### 线程结构

具体讲到kernel函数，运行时指定结构为<<<gridDim, blockDim, sharedMemroySize, stream>>>，其基本是一个网格，网格中有多个线程块，每个线程块中有多个线程，每个线程运行在SP上

* gridDim和blockDim 是Dim3类型，可以简单的视作为（x, y, z）
* gridDim: 网格形状
* blockDim: 线程块形状
* sharedMemroySize: 共享内存大小
* stream: 流

上面说到kernel函数中blockIdx和threadIdx在每个线程中是不一样的，其组合起来确定唯一的线程id呢

* tid = blockId * blockSize + threadId
* blockId 或 threadId 是需要高维降一维，公式为`x + y Dx + z Dx Dy`， 需要注意的是在cuda中[x,y,x], x是低维->高维，而日常习惯中 x是高维到低维
  
  ```cpp
  __global__ void vecAdd(float* A, float* B, float* C) {
    int blockid = blockIdx.y*gridDim.x+blockIdx.x;
    int idx = blockid*(blockDim.x*blockDim.y) + threadIdx.y*blockDim.x + threadIdx.x;
    C[idx] = A[idx] + B[idx];
  }
  ```


### 内存结构

指cuda程序在运行时，GPU上的内存结构

* 每个SP上运行一个cuda线程，每个cuda线程拥有自己的寄存器和localMemroy
* 每个线程块有一个共享内存，线程块中的线程可以访问该共享内存，其背后基于L1级缓存
* 所有的线程都能够读取全局内存

### 工作流

1. host端将需要计算的数据拷贝到device端, 数据：host->device
2. host端调用kernel函数，触发GPU开始运算
3. host端同步device端的数据，数据：device->host
4. 如有需要重复123步骤

### CUDA内部调度

在host执行kernel函数时，cuda程序是如何在GPU运行大量的线程呢

* 一个GPU卡有多个SM，以线程块为单位分发到各个SM上
  * 一个grid有多个thread block 
  * 一个grid下的blocks有可能被调度运行到多个SM上
  * 一个thread block中所有的线程 一定是运行在一个SM上
* 在线程块中，以wrap为单位进行切分，wrap大小是nvidia设置的，默认32，可修改，wrap大小的线程束将被wrap scheduler真正的调度执行
  * wrap大小的线程束将被wrap scheduler真正的调度执行
  * 一个SM中能同时执行的wrap的数量取决于sp的个数、寄存器的数量、共享内存的大小（L1缓存）
  * 线程块切割成wrap的数量为 总线程数量除以wrap大小，有小数的话就向上取整

### 优化方向

#### 数据传输

让数据更快的传输在CPU和GPU之间，比如 page-lock内存块，host端的内存类型有两种

* 普通Memory

* page-locked(pinned) Host memory，加速原因

  ```
  The GPU always must DMA from pinned memory. If you use malloc() for your host data, then it is in pageable (non-pinned memory). When you call cudaMemcpy(), the CUDA driver has to first memcpy the data from your non-pinned pointer to an internal pinned memory pointer, and then the host->GPU DMA can be invoked.
  
  If you allocate your host memory with cudaMallocHost and initialize the data there directly, then the driver doesn’t have to memcpy from pageable to pinned memory before DMAing – it can DMA directly.
  
  That is why it is faster.
  ```

  过多的分配page-locked内存会导致操作系统可用的物理内存变少，可能影响性能

#### kernel函数

根据算法特性优化，比如使用share-memory，share-memory 使用的是L1级缓存，在同一个线程块共享同一个shared-memory, 避免频繁使用 global memory，进而加速，shared_memory 使用`__shared__`关键字声明一块共享内存，这个具体要看kernel函数执行的算法才能加速，例如，向量相乘，加速性能

```cpp
struct Matrix {
  Matrix() = default;
  Matrix(int w, int h) : width(w), height(h), elements(nullptr) {}
  
  int width;
  int height;
  float* elements;
};

__device__ Matrix getSubMatrix(const Matrix mrx, int row, int col) {
  Matrix submrx;
  submrx.width = BLOCKSIZE;
  submrx.height = BLOCKSIZE;
  submrx.elements = &mrx.elements[mrx.width*row*BLOCKSIZE + BLOCKSIZE*col];
  return submrx;
}

__global__ void vecMulWithSharedMemory(const Matrix A, const Matrix B, Matrix C) {
  Matrix Csm = getSubMatrix(C, blockIdx.y, blockIdx.x);
  float val = 0;
  for (int i = 0; i < (A.width/BLOCKSIZE); ++i) {
    Matrix Asm = getSubMatrix(A, blockIdx.y, i);
    Matrix Bsm = getSubMatrix(B, i, blockIdx.y);
    
    __shared__ int As[BLOCKSIZE][BLOCKSIZE];
    __shared__ int Bs[BLOCKSIZE][BLOCKSIZE];
    As[threadIdx.y][threadIdx.x] = Asm.elements[threadIdx.y*BLOCKSIZE+threadIdx.x];
    Bs[threadIdx.y][threadIdx.x] = Bsm.elements[threadIdx.y*BLOCKSIZE+threadIdx.x];
    __syncthreads();

    for (int j = 0; j < BLOCKSIZE; ++j) {
      val += As[threadIdx.y][j] * Bs[j][threadIdx.x];
    }
    __syncthreads();
  }
  C.elements[C.width*blockIdx.y*BLOCKSIZE + BLOCKSIZE*blockIdx.x] = val;
}
```



#### 并发执行

尽可能的kernel不间断的并发运行，如使用异步相关的接口和stream，原理：异架构编程中，一些任务是能够并行执行的，例如

* 简单的内存拷贝在host和device端
* 内存拷贝在device内部
* kernel函数加载执行
* device内存设置赋值memory set

更简单的也可以理解为host的任务和device端的任务是能够并行执行的

##### STREAM

一系列操作的组合的队列，FIFO，该队列的生产者可以是host端，消费者为device端的驱动，host端指定操作放入stream便返回，这样就可以实现host端和device端同时工作。 

* 通过stream来进行管理这一系列的所有并发操作，所有存在依赖关系的操作最好放在同一个stream里面，执行顺序就是放入stream的顺序
* 多stream之间无序的，所以需要，可以手动synchronize

**注意**：放入stream中的内存拷贝（host->device或者device->host）,host端的内存必须是page-lock内存，否则还是隐式的同步，而不是异步



## CUDA驱动相关

独立的操作系统？好像自身除了没有磁盘外，其他都有，在CUDA编程中

1. GPU进程管理
   * cu_context ：类似于进程，资源管理的单位，在cuda runtime任意api调用的使用隐式的创建
   * 进程切换？以cu_context为单位，进行时间切片分时复用GPU
   * 进程执行
     * 一个cpu进程可以在一张卡上创建多个cu_context，但是实际执行的时候，都是分时复用，增加cu_context切换的开销，选择单cu_context多stream更妥当，因为同一context下的多stream可以并发，参考[文档](https://docs.nvidia.com/deploy/mps/index.html#topic_4_1)
2. GPU内存管理
   * 虚拟内存？以什么为分割？也是分页管理显存
   * 内存交换？capability 6.x之后的显卡支持和host交换显显存，类似swap
   * cache： l1,l2两层cache