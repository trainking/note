# golang细节

- [golang细节](#golang细节)
	- [1. 如何理解经典语？](#1-如何理解经典语)
		- [什么是共享内存呢？](#什么是共享内存呢)
		- [通信来共享内存](#通信来共享内存)
	- [2. defer](#2-defer)
		- [defer约法三章](#defer约法三章)
	- [3. Channel](#3-channel)
		- [3.1 关闭channel之后，channel的问题？](#31-关闭channel之后channel的问题)

## 1. 如何理解经典语？

> 不要通过共享内存来通信，而应该通过通信来共享内存

这是`golang`社区经典语句，如何理解这句话呢

### 什么是共享内存呢？

首先要明白，`golang`中携程是没有父子关系的，在一个携程创建另一个携程的时候，只是将所谓**父携程**的内存共享给了**子携程**。**子携程**对**父携程**的变量可以直接引用和修改：

```golang
func main() {
	var aInt = 1

	go func() {
		aInt = 2
	}()

	time.Sleep(3 * time.Second)
	fmt.Println(aInt)
	return
}

// out: 2
```

### 通信来共享内存

上面这段代码中，使用`time.Sleep`来等待携程完成赋值。这样直接就可以共享内存的方式，会带来**线程不安全**的后果，因为我对变量的修改，我并不知道她什么时候完成了，而且如果谁可以修改，那么这个`aInt`什么时候是2，就变得不可靠了，这个时候就需要引用`chan`来通信，告诉程序，完成了修改：

```golang
func main() {
	var aInt = 1
	
    var ch = make(chan bool)
	go func() {
		aInt = 2
		ch <- true
	}()

	<-ch
	fmt.Println(aInt)
	return
}
// out；2
```

而如果将输出语句也移到一个携程之中，则就更加需要`chan`来完成通信了：

```golang
var ch = make(chan bool)
go func() {
    select {
    case <-ch:
        fmt.Print(aInt)
    }
}()

go func() {
    aInt = 2
    ch <- true
}()
```

## 2. defer

`defer`本意是在函数结束时，执行一些必要的步骤。每定义一个`defer`，将压入一个栈中，所以有多个defer时，是**逆序**执行：

```golang
func main() {
	for i := 0; i < 5; i++ {
		defer fmt.Print(i)
	}
}
// out: 43210
```

而`defer`定义时，引用的变量，是不会变的, 如:

```golang
func main() {
	var aInt = 1
	defer fmt.Print(aInt)

	aInt = 2
}
out: 1
```

只取值定义时的值，而如果`defer func(){}`中引用了**父携程变量**，则因为共享内存的机制，会发生改变：

```golang
func main() {
	var aInt = 1
	defer func() {
		fmt.Println(aInt)
	}()

	aInt = 2
}
// out: 2
```

如果作为参数传递，则因为值传递的原因，则不会变:

```golang
func main() {
	var aInt = 1
	defer func(a int) {
		fmt.Println(a)
	}(aInt)

	aInt = 2
}
// out: 1
```

### defer约法三章

`defer`的语法极其简单，但是衍生出来的花里胡哨的操作却极其之多，所以`golang`官方博客总结三条行为准则：

1. 延迟函数的参数在`defer`语句之前已经确定
2. 延迟函数按LIFO(后进先出)的顺序执行（栈操作）
3. 延迟函数可能操作主函数的具名返回值

前两条都好理解，上面的例子已经说明了，第三条是怎么回事呢？

首先要明白一个点，那就是`return`并不是一个**原子操作**，实际上`return`只是代表汇编指令`ret`，只是跳转程序执行，而值是放在返回栈中的。又因为主函数的具名返回值，其实是一个指向返回值空间的引用，所以是可以修改其值的。

```golang
func dd() (result int) {
	i := 1

	defer func() {
		result++
	}()
	return i
}
// return: 2
```

匿名返回值，可以被应用，但是其不可被修改:

```golang
func dd() int {
	i := 1

	defer func() {
		i++
	}()
	return i
}
// return: 1
```

## 3. Channel

### 3.1 关闭channel之后，channel的问题？

* 如果`chan`是无缓存的，则`send`和`recv`都会开始阻塞，直到双方就绪才会执行下一步，所以close之后，再`send`和`recv`都会触发`panic`
* 如果`chan`是有缓存的，则`send`会`paic`，而`recv`可以将缓存区读完，最后返回0值，第二个返回`ok`是false结束
* 未被初始化分配空间的`chan`是nil, `send`和`recv`都会`panic`
