# OpenMP

**OpenMP**是一个非常易用的内存共享并行编程框架，使用一种可移植、可伸缩的模型，提供给编程者一个简单而灵活的接口来开发并行应用，支持多平台共享内存的C、C++、Fortran多处理器编程，可以运行在绝大多数处理器架构和操作系统上，包括Solaris、AIX、HP-UX、GNU/Linux、Mac OS X和Windows平台。

## 快速使用

一个简单的OpenMP实例：

```C
#include <stdio.h>
#include <omp.h>

int main() {

    #pragma omp parallel num_threads(4)
    {
        printf("hello world from tid = %d\n", omp_get_thread_num());
    }
    return 0;
}
```

编译时，需要在`gcc`命令后加上`-fopenmp`，指定使用OpenMP框架。

## 基本原理

在**OpenMP**的程序当种，可以将程序并行分开，在并行域中，程序是并发的，但是在并行域之外是没有并发的，只有主线程的在执行。