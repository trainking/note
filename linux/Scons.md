# Scons

- [Scons](#scons)
  - [安装](#安装)
  - [使用](#使用)
    - [1. 编译一个文件](#1-编译一个文件)
    - [2. 编译多个文件](#2-编译多个文件)
    - [3. 编译时使用Lib](#3-编译时使用lib)
    - [4. 清楚编译缓存](#4-清楚编译缓存)
  - [特别注意](#特别注意)

**Scons** 是一个python编写的构建工具，在学习godot引擎时发现这个工具。使用起来发现的确很好用，记录此份笔记。

## 安装

安装过程，直接使用pip安装就好了：

```sh
sudo pip install scons
```

## 使用

**Scons** 使用的配置文件叫`Sconstruct`，本质是一个python的脚本。

### 1. 编译一个文件

```python
# Sconstruct

Program('main.cpp')
```

### 2. 编译多个文件

```python
# Sconstruct


Program('taskpool', ['TaskPool.cpp', 'main.cpp'])
```

### 3. 编译时使用Lib

```python
# Sconstruct

Program('taskpool', ['TaskPool.cpp', 'main.cpp'], LIBS=['pthread'])
```

### 4. 清楚编译缓存

```
scons -c
```

## 特别注意