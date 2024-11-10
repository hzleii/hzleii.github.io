---
title: 'go：原子操作 atomic'
date: 2023-10-11T21:48:11+08:00
pubdate: 2023-10-11T21:48:11+08:00
keywords:
  - 'go'
  - 'atomic'
  - '教程'
  - '指南'
description: 'go语言中的原子操作（atomic）。'
---


## 1.基础

关于原子操作，百度百科是这么解释的：

> 所谓原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch（切换到另一个线程）。

也就是在其他线程看来，原子操作要么执行完了，要么没有执行，不会看到执行一半的结果。

CPU 提供了基础的原子操作，但不同架构的系统的原子操作是不一样的。

在单处理器单核系统中，CPU 在任何时刻只能执行一个指令，如果一个操作可以由单个 CPU 指令完成，那么这个操作就是原子的，例如 XCHG（交换指令，用于交换两个寄存器或内存位置的值）和 INC（增加指令，用于将寄存器或内存位置的值增加 1）就是原子操作。如果一个操作是由多条指令完成的，那么在执行过程中就可能会被中断，并执行上下文切换，就不是原子操作。

多处理器多核系统会比较复杂，由于缓存的存在，当单个核上的单个指令执行原子操作时，必须确保其他处理器或核无法访问该原子操作涉及的内存地址，或者确保它们总是能够访问到原子操作完成后的最新数据。

在 x86 架构中，通过使用 LOCK 指令前缀，可以确保像 LOCK CMPXCHG（比较并交换）这样的指令在执行时不会被其他处理器或核干扰。此外，某些指令如 XCHG，本身就具备锁定功能。

不同的 CPU 架构采用不同的方法来实现原子操作，例如，对于支持多核的 MIPS 和 ARM 架构，它们提供了LL/SC（Load Link/Store Conditional）指令，这些指令能够支持原子操作的实现。

> LL（Load Linked，链接加载）：
> 
> LL 指令用于从内存中读取一个字（或数据）并将其加载到寄存器中。同时，LL 指令还会在处理器内部设置一个不可见的标记，用来跟踪这个内存地址的状态。这个操作确保了后续的 SC 指令能够检查在LL指令执行之后，是否有其他处理器对这个内存地址进行了修改。
> 
> SC（Store Conditional，条件存储）：
> 
> SC 指令用于将寄存器中的值有条件地写回到之前 LL 指令读取的内存地址。SC 指令会检查自从 LL 指令执行以来，内存地址是否被其他处理器修改过。如果内存地址未被修改，SC 指令会将寄存器中的值写入内存，并将寄存器的值设置为1表示操作成功。如果内存地址被修改过，SC 指令将放弃写操作，并将寄存器的值设置为 0 表示操作失败。
> 
> 原子性保证：
> 
> LL/SC 指令对的执行确保了读-改-写（RMW）操作的原子性。如果 SC 指令成功执行，那么可以确信在 LL 和 SC 之间没有其他线程或处理器对这个内存地址进行了干扰。
> 
> 不同架构的支持：
> 
> 不同的 CPU 架构以不同的方式实现 LL/SC 指令。例如，在 MIPS 和 ARM 架构中，LL/SC指令分别以 ll/sc 和 ldrex/strex 的形式存在，用于支持原子操作。

虽然很复杂，涉及到不同系统不同架构，但好在 Go 提供了一个通用的原子操作 API，已经帮我们封装成 atomic 包。

接下来我们用汇编来体验一下。

```go
func main() {
    i := 0
    i++
}
```

我们保留需要的汇编代码

```sh
$ go tool compile -S -N main.go
......
0x000c 00012 (main.go:4)  MOVD    ZR, main.i-8(SP)    # 将零寄存器（ZR）的值移动到栈指针（SP）减去 8 的位置，即局部变量 i 的存储位置。
0x0010 00016 (main.go:5)  MOVD    $1, R0              # 将立即数 1 移动到寄存器 R0 中。这里的 $1 表示数值 1。
0x0014 00020 (main.go:5)  MOVD    R0, main.i-8(SP)    # 将寄存器 R0 的值（即 1）移动到栈指针减去 8 的位置，也就是局部变量 i 的存储位置。也就是完成 i++ 操作。
.....
```

我们再使用 atomic 来查看汇编代码。

```go
package main

import (
	"sync/atomic"
)

func main() {
	var i int64 = 0
	atomic.AddInt64(&i, 1)
}
```

```sh
$ go tool compile -S -N main.go
......
0x0018 00024 (main.go:8)  MOVD    $type.int64(SB), R0     # 将 int64 类型的类型信息的地址（在静态数据区）加载到寄存器 R0 中。这是为创建一个新的 int64 类型的变量准备。
0x0020 00032 (main.go:8)  CALL    runtime.newobject(SB)   # 调用 runtime.newobject 函数，这个函数负责在堆上分配一个 int64 大小的内存块，并返回其地址。
0x0024 00036 (main.go:8)  MOVD    R0, main.&i-8(SP)       # 将 runtime.newobject 返回的地址（即新分配的 int64 变量的地址）存储在栈上，具体位置是 main.&i-8(SP)，这是局部变量 i 的地址。
0x0028 00040 (main.go:8)  MOVD    ZR, (R0)                # 将零寄存器（ZR）的值（即 0）移动到由 R0 寄存器指向的内存地址，这里 R0 是 i 的地址，所以这行代码将 i 初始化为 0。
0x002c 00044 (main.go:9)  MOVD    main.&i-8(SP), R1       # 将局部变量 i 的地址加载到寄存器 R1 中。
0x0030 00048 (main.go:9)  MOVBU   runtime.arm64HasATOMICS(SB), R2     # 将 runtime.arm64HasATOMICS 的值（一个布尔值，指示 ARM64 是否支持原子操作）加载到寄存器 R2 中。
0x003c 00060 (main.go:9)  TBNZ    ZR, R2, 68              # 如果 R2（即 runtime.arm64HasATOMICS）不为零，则跳转到地址 68（即第 9 条指令）。
0x0040 00064 (main.go:9)  JMP     84                      # 如果不支持原子操作，则跳转到地址 84（即第 13 条指令）。
0x0044 00068 (main.go:9)  MOVD    $1, R0                  # 将立即数 1 加载到寄存器 R0 中。
0x0048 00072 (main.go:9)  LDADDALD        R0, (R1), R2    # 如果支持原子操作，使用 LDADDALD 指令将 R0（即 1）加到 R1 指向的地址（即 i 的值），并将 R1 的原始值存储在寄存器 R2 中。
0x004c 00076 (main.go:9)  ADD     R0, R2                  # 将 R0（即 1）加到 R2 中。
0x0050 00080 (main.go:9)  JMP     108                     # 跳转到地址 108，结束这个代码块。
0x0054 00084 (main.go:9)  MOVD    $1, R0                  # 如果不支持原子操作，将立即数 1 加载到寄存器 R0 中。   
0x0058 00088 (main.go:9)  LDAXR   (R1), R2                # 使用 LDAXR 指令从 R1 指向的地址（即 i 的值）加载值到 R2 中。
0x005c 00092 (main.go:9)  ADD     R0, R2                  # 将 R0（即 1 ）加到 R2 中。
0x0060 00096 (main.go:9)  STLXR   R2, (R1), R27           # 使用 STLXR 指令尝试将 R2 的值存储回 R1 指向的地址，并在 R27 中存储操作的结果。
0x0064 00100 (main.go:9)  CBNZ    R27, 88                 # 如果 R27 不为零，说明 STLXR 操作失败，跳转到地址 88（即第 14 条指令）。
0x0068 00104 (main.go:9)  JMP     108                     # 如果 STLXR 操作成功，跳转到地址 108，结束这个代码块。
......
```

可以看到，它会判断当前 ARM 架构是否支持原子操作，如果不支持，则使用 ARM 的 LL/SC 指令 LDREX 和 STREX。

## 2.方法

我们以 [go1.23.2](https://github.com/golang/go/blob/go1.23.2/src/sync/atomic/doc.go "https://github.com/golang/go/blob/go1.23.2/src/sync/atomic/doc.go") 来介绍 atomic 提供的 API。

**atomic 操作的对象是一个地址，所以需要把可寻址的变量的地址作为参数传递给方法，而不是把变量的值传递给方法**。

### 2.1 Swap 交换值

用于原子地交换指定内存地址的值，并返回原来的值。伪代码如下：

```ini
old = *addr
*addr = new
return old
```

支持的数据类型和方法：

```go
func SwapInt32(addr *int32, new int32) (old int32)

func SwapInt64(addr *int64, new int64) (old int64)

func SwapUint32(addr *uint32, new uint32) (old uint32)

func SwapUint64(addr *uint64, new uint64) (old uint64)

func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)

func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)
```

### 2.2 CompareAndSwap（CAS）

比较旧值，如果相等就替换。伪代码如下：

```kotlin
if *addr == old {
	*addr = new
	return true
}
return false
```

支持的数据类型和方法：

```go
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)

func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)

func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)

func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)

func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)

func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)
```

### 2.3 Add

用于原子地对指定的内存地址的值进行加法操作，并**返回计算后的值**。伪代码如下：

```sql
old = *addr
*addr = new+old
return old
```

支持的数据结构和方法：

```go
func AddInt32(addr *int32, delta int32) (new int32)

func AddUint32(addr *uint32, delta uint32) (new uint32)

func AddInt64(addr *int64, delta int64) (new int64)

func AddUint64(addr *uint64, delta uint64) (new uint64)

func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)
```

#### 2.3.1 AddUint 方法如何做减法？

1.  定义有符号临时变量，通过类型转换为无符号值：

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var old uint32 = 10
	var delta int32 = -2
	n := atomic.AddUint32(&old, uint32(delta))

	fmt.Println(n)
}
```

2.  将差量减 1 再取反

```go
func main() {
	var old uint32 = 10
	var delta int32 = 2
	n := atomic.AddUint32(&old, ^uint32(delta-1))

	fmt.Println(n)
}
```

### 2.4 And（按位与）

用于原子地对指定的内存地址的值进行按位与操作，并**返回原来的值**，旧版本不支持。

```go
func AndInt32(addr *int32, mask int32) (old int32)

func AndUint32(addr *uint32, mask uint32) (old uint32)

func AndInt64(addr *int64, mask int64) (old int64)

func AndUint64(addr *uint64, mask uint64) (old uint64)

func AndUintptr(addr *uintptr, mask uintptr) (old uintptr)
```

### 2.5 Or（按位或）

用于原子地对指定的内存地址的值进行按位或操作，并**返回原来的值**，旧版本不支持。

```go
func OrInt32(addr *int32, mask int32) (old int32)

func OrUint32(addr *uint32, mask uint32) (old uint32)

func OrInt64(addr *int64, mask int64) (old int64)

func OrUint64(addr *uint64, mask uint64) (old uint64)

func OrUintptr(addr *uintptr, mask uintptr) (old uintptr)
```

### 2.6 Load

从内存地址 addr 中读取值并返回。

```go
func LoadInt32(addr *int32) (val int32)

func LoadInt64(addr *int64) (val int64)

func LoadUint32(addr *uint32) (val uint32)

func LoadUint64(addr *uint64) (val uint64)

func LoadUintptr(addr *uintptr) (val uintptr)

func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
```

### 2.7 Store

Store 会把一个值原子地存入到指定的 addr 地址中，就算有别的 goroutine 通过 Load 读出来，也不会看到存取了一半的值。

```go
func StoreInt32(addr *int32, val int32)

func StoreInt64(addr *int64, val int64)

func StoreUint32(addr *uint32, val uint32)

func StoreUint64(addr *uint64, val uint64)

func StoreUintptr(addr *uintptr, val uintptr)

func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
```

## 3 Value 类型

atomic 还提供了一个特殊的类型 `Value`。它可以原子地 Load, Store, Swap, CAS 任意类型。

举个例子：

```go
package main

import (
	"fmt"
	"sync/atomic"
)

type User struct {
	Id   string
	Name string
	Age  int
}

func main() {
	var v atomic.Value
	userA := User{
		Id:   "id1",
		Name: "name1",
		Age:  1,
	}
	userB := User{
		Id:   "id2",
		Name: "name2",
		Age:  2,
	}
	userC := User{
		Id:   "id3",
		Name: "name3",
		Age:  3,
	}

	v.Store(userA)
	fmt.Println(v) // {{id1 name1 1}}

	old := v.Swap(userB)
	fmt.Println(old)        // {id1 name1 1}
	fmt.Println(old.(User)) // {id1 name1 1}

	swapped := v.CompareAndSwap(userB, userC)
	fmt.Println(swapped, v) // true {{id3 name3 3}}

	swapped = v.CompareAndSwap(userA, userB)
	fmt.Println(swapped, v) // false {{id3 name3 3}}
}
```

使用 `atomic.Value` 需要注意以下几点：

1.  不能存 nil，会 panic

```go
var v atomic.Value
v.Store(nil)    // panic: sync/atomic: store of nil value into Value
```

2.  相同的 Value 变量前后存入的数据类型要一致，否则 panic

```go
var v atomic.Value
v.Store("user")
v.Swap(1)   // panic: sync/atomic: swap of inconsistently typed value into Value
```

3.  最好别存引用类型的值，会导致数据一致性问题

```go
package main

import (
	"fmt"
	"sync/atomic"
)

type m map[int]int

func main() {
	var v atomic.Value
	mm := make(m)

	v.Store(mm)

	m1 := v.Load().(m)
	m1[1] = 200

	m2 := v.Load().(m)
	fmt.Println("m2 len: ", len(m2)) // 1
	fmt.Println("m2[1]: ", m2[1])    // 200
}
```

## 4.小结

本文我们讲述了原子操作以及 Go 语言封装的 atomic 包，相比互斥锁更加轻便，希望对你有帮助。
