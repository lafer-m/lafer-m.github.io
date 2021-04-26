---
layout: post
title: golang/unsafe及string详解
tags: [golang]
---

unsafe.Pointer的使用规则及string的底层结构.

#### unsafe包的使用及string的内存布局
golang中unsafe包的使用规则，
* 任意类型的指针，都可以转换为unsafe.Pointer；
* unsafe.Pointer可以转换为任意类型的指针；
* uintptr及unsafe.Pointer可以互相转换；  

一些使用场景：  
* *T1 转换为 *T2  需要相同的内存布局

```
// math.Float64bits
func Float64bits(f float64) uint64 {
		return *(*uint64)(unsafe.Pointer(&f))
}
```

* uintptr与Pointer的转换
string在底层是一个简单的结构体，维护了指向底层数组的指针及数据长度，如下例子中进行的了详细的解析。

```
package main

import (
	"fmt"
	"unsafe"
	"encoding/binary"
)

func main() {
	a := [4]byte{'a','b','c','d'}
	fmt.Printf("%#v\n",unsafe.Pointer(&a))
	
	// 模拟的string的内存结构，实际上golang的实现是两个字段，第一个是数据指针，第二个是长度。
	type strHeader struct {
		Data uintptr
		Len int	
	}
	str := &strHeader{
		Data: uintptr(unsafe.Pointer(&a)),
		Len: 4,
	}
	s1 := (*string)(unsafe.Pointer(str))
	fmt.Printf("%s\n", *s1)

	//str1 := "abcd"
	memoryLayout := (*[16]byte)(unsafe.Pointer(s1))
	fmt.Printf("%#v\n", memoryLayout)
	
	// 从内存结构中获取到数据指针的值
	dataPtr := memoryLayout[:8]
	
	// 获取到这个unsafe指针
	ptr := unsafe.Pointer(uintptr(binary.LittleEndian.Uint64(dataPtr)))
	fmt.Printf("%#v\n",ptr)
	
	// 通过指针取数组a的值， unsafe指针可以转换为任意类型的指针
	fmt.Printf("%s", *(*[4]byte)(ptr))
	
}
```
&nbsp; &nbsp; 初看代码是没有啥问题的，也能够正常打印出我们想要的结果，我们可以通过构造的unsafe pointer转换为golang的string指针，然后在通过指针拿到字符串的值；同样反过来操作也是可以的，通过字符串指针，拿到数据数组的指针，取到数组的值，这个操作能够成功，是因为数组并没有被gc或者移动内存地址，而gc是golang的运行时特性，所以无法保证指针的可用性。  

&nbsp; &nbsp; 如果用go vet .检查，发现会有错误出现`./prog.go:33:9: possible misuse of unsafe.Pointer` 为了理解这个错误，我们必须理解unsafe.Pointer的规则限制:

```
Converting a Pointer to a uintptr produces the memory address of the value pointed at, as an integer. The usual use for such a uintptr is to print it.

Conversion of a uintptr back to Pointer is not valid in general.

A uintptr is an integer, not a reference. Converting a Pointer to a uintptr creates an integer value with no pointer semantics. Even if a uintptr holds the address of some object, the garbage collector will not update that uintptr’s value if the object moves, nor will that uintptr keep the object from being reclaimed.
```

** 转换一个数据指针为uintptr的（int）值通常只是为了打印这个数据指针；   
** 将一个uintptr转换回一个数据指针通常是无效的；   
** 一个unintptr是个整数，不是引用地址，创建一个uintptr只是创建了一个整数数值，并没有指针的语意；即使一个uintptr值是相同object的内存地址，gc机制也不会更新uintptr的值，当object移动或者被重新赋值的时候。

&nbsp; &nbsp; 理解了上述的规则也就明白了何时能够正确的使用unsafe.Pointer和uintptr了，只有当能够确定某个object不会被gc影响的时候，才能保证是安全的。unsafe包也有对安全使用的描述,最通用的使用场景如取struct或者数组的某个filed的值。
* 不能将uintptr保存到变量中，但是阔以对它进行计算然后马上转换为pointer.

```
If p points into an allocated object, it can be advanced through the object
by conversion to uintptr, addition of an offset, and conversion back to Pointer.
	p = unsafe.Pointer(uintptr(p) + offset)
```

不像c语言中，指针操作是可以越界访问内存的，golang中这种操作是不合法的，如下代码，访问的内存地址超过了s申请的内存大小：

```
var s thing  
end = unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Sizeof(s))  
b := make([]byte, n)  
end = unsafe.Pointer(uintptr(unsafe.Pointer(&b[0])) + uintptr(n))
```

* 当uintptr作为参数的时候，编译器保证调用期间转换为pointer可用。  
&nbsp; &nbsp; 当调用syscall.Syscall的时候，需要传uintptr值，调用的过程中会将uintptr转换为pointer；   
syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(p)), uintptr(n))  
go编译器可以处理这种在参数中的uintptr，会保证在调用期间，object不会被移动或者回收  

* reflect.SliceHeader和reflect.StringHeader
&nbsp; &nbsp; reflect.SliceHeader和reflect.StringHeader的Data项就是uintptr，这两个struct为什么可以安全的使用了。文档也解释了

```
var s string
hdr := (*reflect.StringHeader)(unsafe.Pointer(&s))
hdr.Data = uintptr(unsafe.Pointer(p))              // 这种情况下hdr.Data实际上是对s header的数据引用，而不是uintptr本身的值；而且这两个struct只能用做*指针，不能用struct value
hdr.Len = n
```

* reflect.Value.Pointer 或者 reflect.Value.UnsafeAddr 转换为pointer   
这个只能取到uintptr后，马上转换为Pointer,不能存为变量，如
p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))。




#### 例子：

```
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	arr := []uint32{1, 2, 3}
	ptr := uintptr(unsafe.Pointer(&arr[0]))
	arr = []uint32{2,2,3}
	fmt.Printf("%d \n", (*(*uint32)(unsafe.Pointer(ptr))))
	fmt.Printf("%d \n", arr[0])
}

```
这里能够取到老的值(内存地址的值并没有被改变)；这样会导致安全问题，假如是一个http接口使用了unsafe指针uintptr，而这个指针地址的值存储了用户密码等信息。
