## 一 同步解决方案

Go程序可以使用通道进行多个goroutine间的数据交换，但是这仅仅是数据同步中的一种方法，通道内部的实现依然使用了各种锁，因此优雅代码的代价是性能。  

在某些轻量级的场合，原子访问（sync/atomic包），互斥锁（sync.Mutex）以及等待组（sync.WaitGroup）能最大程度满足需求。  

## 二 锁

## 1.1 互斥锁 sync.Mutex

互斥锁是传统并发程序进行共享资源访问控制的主要方法。  

Go中由标准库`sync`中的结构体`Mutex`表示互斥锁，保证同时只有一个 goroutine 可以访问共享资源，该结构体包含两个方法：
```
Lock：锁定当前互斥量
Unlock：解锁当前互斥量
```

使用方式：
```go
var mutex sync.Mutex
func write() {
	mutex.Lock()
	defer mutex.Unlock()
	// 业务逻辑....
}
```

使用案例：
```go
package main

import (
	"fmt"
	"sync"
)

var count int
var countGuard sync.Mutex			// 变量对应的互斥锁

func GetCount() int {
	countGuard.Lock()				// 加锁
	defer countGuard.Unlock()		// 函数退出时解锁
	return count
}

func SetCount(i int) {
	countGuard.Lock()
	count = i
	countGuard.Unlock()
}

func main() {

	SetCount(1)				// 并发安全的设置值
	fmt.Println(GetCount())		// 并发安全的获取值

}
```

在上述案例中 一旦 countGuard发生加锁，如果另外一个 goroutine尝试继续加锁时将会发生阻塞，直到这个 countGuard被解锁，所以在使用互斥锁时应该注意一些常见情况：  
- 对同一个互斥量的锁定和解锁应该承兑出现，对一个已经锁定的互斥量进行重复锁定，会造成goroutine阻塞，直到解锁，下列给出了一个具体案例
- 对未加锁的互斥锁解锁，会引发运行时崩溃，1.8版本之前可以使用defer可以有效避免该情况，但是重复解锁容易引起goroutine永久阻塞，1.8版本之后无法利用defer+recover恢复

具体案例：
```go
import (
	"fmt"
	"sync"
	"time"
)

func main() {

	var mutex sync.Mutex
	fmt.Println("Lock.....")
	mutex.Lock()

	for i := 1; i <=3; i++ {
		go func(i int) {
			fmt.Println("当前 i：", i)
			mutex.Lock()
			fmt.Println("循环中Lock i：", i)
		}(i)
	}

	time.Sleep(time.Second)
	mutex.Unlock()
	fmt.Println("Unlock main....")
	time.Sleep(time.Second)
}
```  

输出结果如下：
```
Lock.....
当前 i： 3
当前 i： 2
当前 i： 1
Unlock main....
循环中Lock i： 3		// 协程阻塞结束，获得锁，但此时循环已经结束了，所以只有一次打印，且必定是最后一个i
```

#### 2.2 读写锁 sync.RWMutex

读写锁即针对读写操作的互斥锁。与普通锁的区别是：可以分别针对读操作和写操作进行锁定和解锁操作。  

读写锁的访问控制规则与互斥锁有所不同：
- 读写锁控制下的多个写操作之间是互斥的
- 写操作与读操作之间也是互斥的
- 多个读操作之间不存在互斥关系

在Go中，读写锁由结构体`sync.RWMutex`表示，包含两对方法：
```go
// 下列两个方法与2.1章节互斥锁使用方式一致
func (*RWMutex) Lock()				// 锁定写
func (*RWMutex) Unlock()			// 解锁写

// 对读执行加锁解锁
func (*RWMutex) RLock()
func (*RWMutex) RUnlock()
```
注意：
- Mutex和RWMutex都不关联goroutine，但RWMutex显然更适用于读多写少的场景。仅针对读的性能来说，RWMutex要高于Mutex，因为rwmutex的多个读可以并存。
- 所有被读锁定的goroutine会在写解锁时唤醒
- 读解锁只会在没有任何读锁定时，唤醒一个要进行写锁定而被阻塞的goroutine
- 对未被锁定的读写锁进行写解锁或读解锁，都会引发运行时崩溃
- 对同一个读写锁来说，读锁定可以有多个，所以需要进行等量的读解锁，才能让某一个写锁获得机会，否则该goroutine一直处于阻塞，但是sync.RWMutext没有提供获取读锁数量方法，这里需要使用defer避免，如下案例所示。

```go
import (
	"fmt"
	"sync"
	"time"
)

func main() {

	var rwm sync.RWMutex
	
	for i := 0; i < 3; i++ {
		go func(i int) {
			fmt.Println("Try Lock reading i:", i)
			rwm.RLock()
			fmt.Println("Ready Lock reading i:", i)
			time.Sleep(time.Second * 2)
			fmt.Println("Try Unlock reading i: ", i)
			rwm.RUnlock()
			fmt.Println("Ready Unlock reading i:", i)
		}(i)
	}

	time.Sleep(time.Millisecond * 100)
	fmt.Println("Try Lock writing ")
	rwm.Lock()
	fmt.Println("Ready Locked writing ")
}
```

上述案例中，只有循环结束，才会执行写锁，所以输出如下：
```go
...
Ready Locked writing		// 总在最后一行
```

#### 2.3 读写锁补充 RLocker方法

`sync.RWMutex`类型还有一个指针方法`RLocker`：

```go
func (rw *RWMutex) RLocker() Locker
```

返回值Locker是实现了接口`sync.Lokcer`的值，该接口同样被 `*sync.Mutex`和`*sync.RWMutex`实现，包含方法：`Lock`和`Unlock`。  

当调用读写锁的RLocker方法后，获得的结果是读写锁本身，该结果可以调用Lock和Unlock方法，和RLock，RUnlock使用一致。

#### 2.4 最后的说明

读写锁的内部其实使用了互斥锁来实现，他们都使用了同步机制：信号量。