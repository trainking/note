# FlatBuffers

`FlatBuffers`是一个开源高效的序列化工具库，与`ProtocalBuffers`类似，同样需要`IDL`文件，提供了`C++, C#, C, Go, Java, PHP, Python，Javascript`的接口。

最初是为`Android`游戏和注重性能的应用而开发，最大特点是**反序列化**速度极快，或者可以说无需解码。

## FlatBuffers的优势

* **无需反序列化即可访问序列化数据** - FlatBuffers 的不同之处在于它在平面二进制缓冲区中表示分层数据，这样它仍然可以直接访问而无需解析/解包，同时仍然支持数据结构演化（向前/向后兼容）。
* **优秀的内存效率和速度** - 

## 编译器下载安装

- [flatc](https://github.com/google/flatbuffers/releases/tag/v2.0.0)