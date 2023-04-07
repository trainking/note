# Recast & Detour

## 1. 概述

* 项目地址： https://github.com/recastnavigation/recastnavigation
* 参考文档： http://recastnav.com/

### Recast
**Recast**是先进的游戏导航网格构建工具，它可以快速地将**任何几何体**生成一套导航网格。Recast通过一个多步骤地栅格化过程，构建一个导航网格：

1. 首先，Recast通过将三角形栅格化为多层高度场，对输入的三角形网格进行**体素**化
2. 通过应用简单的体素数据过滤器，将角色无法移动的区域的体素去除
3. 由体素网格描述的可行走区域然后被划分为一组二维多边形区域
4. 导航多边形是通过对生成的二维多边形区域进行三角连接和缝合而产生的

### Detour

**Detour**是一个寻路和空间推理工具包，可以将任何导航网格与**Detour**结合使用，但**Recast**生成地最佳。**Detour**提供了一种简单的静态导航网格数据表示，适用于许多简单的情况。它还提供了一个平铺的导航网格表示，允许您在玩家在世界中前进时传入和传出导航数据流，并随着世界的变化重新生成导航网格数据的各个部分。

## 2. 使用教程

### 先从RecastDemo开始

**RecastDemo**是一个全面的演示项目，它是一个厨房水槽的演示，展示了库的所有功能，新手可以查看Sample_SoloMesh.cpp来开始构建navmeshes，以及NavMeshTesterTool.cpp来看看Detour如何被用来寻找路径。

#### 构建RecastDemo

**Windows**

1. 下载SDL开发库，到`RecastDemo\Contrib`目录下，保证`RecastDemo\Contrib\SDL\lib\x86`有值。[下载地址](https://github.com/libsdl-org/SDL/releases)，注意下载的是`SDL2-devel-x.x.x-VC.zip`包。
2. 安装`premake5`，并进入`RecastDemo`目录下，执行`premake5 vs2022`。
3. 通过`Build\vs2022\recastnavigation.sln`，打开项目。
4. 重新生成，并启动项目`RecastDemo`。


