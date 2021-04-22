<!-- # WhyThis
This repo will contain all my questions about system designing and my opinions about them. I'll try to update it everyday. :-)

C++
===

为什么有了`NULL`，`C++11`又引入了`nullptr`?
----
专门用来区分0和空指针。所以很明显的，空指针与0绝对是不能够画上等号的。



为什么`C++`要引入`constexpr`
----

为了解决C++标准中数组的长度必须是一个常量表达式的问题。



为什么`C++11`引入了`auto`关键字？
-----

使用`auto`进行类型推导，其中用在迭代器中是最常见的。


为什么`C++`中的`Lambda`表达式的参数列表可以用`auto`但是普通函数不行？
-----

对于普通函数来说如果将`auto`放到参数列表中去的话，这样的写法会与模版的功能产生冲突。比如声明了`a(T)`和`a(auto)`，这样会造成冲突。



为什么有了`move`之后`C++`又引入了`forward`?
----
[参考](https://zhuanlan.zhihu.com/p/55856487)


为什么要引入`std::function`？
-----



OS
=====
### 为什么需要父进程来回收子进程的资源，子进程不能自己回收吗？
回收资源需要执行代码，执行代码需要内核栈，子进程不可能在执行回收代码的时候把正在使用的内核栈释放了。


条件变量为什么需要加锁和循环
--------


- 加锁是为了保护共享变量读写互斥，比如 `count` , `flag` 
- `wait` 设计成要传入锁是为了防止lost wakeup问题
- 循环是为了防止虚假唤醒问题

#### lost wakeup

`condition_variable` 的 `wait` 需要传入锁的设计就是为了避免lost wakeup
先假定wait不需要锁，看看会出现什么问题

```cpp
V()
{
   acquire(&lock);
   count += 1;
   cond.notify_all();
   release(&lock);
}
void
P()
{
  while(count == 0) {
  	cond.wait();
  }
  acquire(&lock);
  count -= 1;
  release(&lock);
}
```

假设 `P` 正处在执行 `wait` 的过程中， `wait` 由很多汇编代码组成，并不是一个原子的过程，而此时 `V` 调用了 `notify()` ， `notify()` 的原理是去 `cond` 的队列里找 `wait` 在它上面的线程，并且逐一唤醒。由于 `P` 的 `wait` 执行在一半，尚无进入这个队列，它不会被唤醒。这就产生了问题： `P` `wait` 在一个已经被 `notify` 过的 `V` 上。

```cpp
V()
{
   acquire(&lock);
   count += 1;
   cond.notify_all();
   release(&lock);
}
void
P()
{
  acquire(&lock);
  while(count == 0) {
  	cond.wait();
  }
  count -= 1;
  release(&lock);
}
```

一个想法是这样修改，这样就保证了 `P` 检查 `count` 和 `P` 加入 `cond` 的等待队列是一个原子的过程，但是这样会产生死锁， `P` 先运行，并且获得了锁， `V` 就一直被阻塞，导致死锁。
因此最后有了这样的设计

```cpp
V()
{
   acquire(&lock);
   count += 1;
   cond.notify_all();
   release(&lock);
}
void
P()
{
  acquire(&lock);
  while(count == 0) {
  	cond.wait(&lock);
  }
  count -= 1;
  release(&lock);
}
```

在 `cond.wait` 中，把当前线程/进程加入等待队列，并进入睡眠作完成后就会解锁。 
#### 

#### 虚假唤醒

考虑单生产者多消费者的情况，假设可用产品数目为 `count` ，生产者刚生产出一个产品， `count` 为1，然后调用了 `notify_all` ，这时所有等待队列上的进程/线程都会被变成就绪态，但是产品只有一个，第一消费者使用完产品后 `count` 变回了0，但是剩下的消费者还是会被唤醒，这被称为虚假唤醒，如果不放在 `while` 循环里检查一下 `count` 是不是0，就会导致明明没有产品了，消费者还在消费，把产品数变成了负数。
除此之外，如果不写 `while` ，可能会出现消费者比生产者先运行而错误消费的情况。

#### C++中条件变量的使用

生产者

```cpp
 // send data to the worker thread
{
    std::lock_guard<std::mutex> lk(m);
    ready = true;
    std::cout << "main() signals data ready for processing\n";
}
cv.notify_one();
```

消费者

```cpp
 std::unique_lock<std::mutex> lk(m);
cv.wait(lk, []{return ready;});

// after the wait, we own the lock.
std::cout << "Worker thread is processing data\n";
```

`cv.wait` 中传入lambda表达式实际就是下述代码的语法糖

```
while (!ready) wait(lk);
```


### 如何实现锁

#### 自旋锁

自旋锁的实现有三个关键点

- 中断
- test_and_set/compare_and_swap
- memory_barrier



- 首先来看中断，一般自旋锁的实现在 `aquire` 中会关中断，在 `release` 中会重新打开中断，这是因为线程成功获取一把锁后，中断后的代码依然有可能需要获取同一把锁，这样就会死锁。
- test_and_set/compare_and_swap原子操作，如果这个过程不是原子操作的话可能会导致临界区被多个线程/进程进入
- memory_barrier，在 `aquire` 中是为了保证临界区的代码始终要在获取锁之后才执行；在 `release` 中是为了保证临界区的代码在释放锁之前已经执行完毕。

xv6中的实现

```cpp
// Acquire the lock.
// Loops (spins) until the lock is acquired.
void
acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk))
    panic("acquire");

  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}


// Release the lock.
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  __sync_lock_release(&lk->locked);

  pop_off();
}
```


#### 睡眠锁

xv6中的实现

```cpp
void
acquiresleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  while (lk->locked) {
    sleep(lk, &lk->lk);
  }
  lk->locked = 1;
  lk->pid = myproc()->pid;
  release(&lk->lk);
}

void
releasesleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  lk->locked = 0;
  lk->pid = 0;
  wakeup(lk);
  release(&lk->lk);
}
```

睡眠锁内部还有一把自旋锁，来保护自己的数据结构。
这里的代码非常像条件变量，加锁和while循环的原因都与条件变量相同，可以认为这里的条件是**锁是否可以获取**    
`sleep` 中把让本进程睡眠，并把 `PCB` 中的 `chan` 指定为 `lk`   
`wakeup` 中遍历所有进程，找出 `PCB` 中 `chan` 为 `lk` 的进程，并把状态改为就绪态   


为什么操作系统需要锁？
-----

因为应用程序想要使用多个CPU核，但是操作系统又有很多的全局变量，所以如果多个应用程序并行进行系统调用，为了保证正确性，我们是不能够让这些公用的数据结构并行进行修改的。


为什么自旋锁需要关中断？
-----

因为自旋锁需要解决两种情况，第一种是不同CPU之间的并发，同时还需要解决相同CPU上面的中断（如果中断之后的程序也需要获得一样的锁那么就会发生死锁）和普通程序之间的并发。



为什么需要有POSIX？
-----

一般情况下，应用程序通过应用编程接口(API)而不是直接通过系统调用来编程。那么为了完成同一功能，不同内核提供的系统调用（也就是一个函数）是不同的，例如创建进程，linux下是fork函数，windows下是creatprocess函数。我现在在linux下写一个程序，用到fork函数，那么这个程序该怎么往windows上移植？我需要把源代码里的fork通通改成creatprocess，然后重新编译。posix标准的出现就是为了解决这个问题。linux和windows都要实现基本的posix标准，linux把fork函数封装成posix_fork（举例），windows把creatprocess函数也封装成posix_fork，都声明在unistd.h里。这样，程序员编写普通应用时候，只用包含unistd.h，调用posix_fork函数，程序就在源代码级别可移植了。

这种实现又是加了一层中间层来实现功能的。

## Compiler


## Database








Appendix
=====
`C++11`新特性
-----

* 列表初始化
* 




`C++17`新特性
-----

* 结构化绑定
* `if/switch`变量声明强化
*  -->

Anyone who wants to join us, contact me and look forward for your contribution.
-------
-------


我们大概可以按照下面的顺序添加内容：
在该文件`README.md`下找到你的问题的相关课题，比如，我想创建一个问题叫做`为什么需要父进程来回收子进程的资源，子进程不能自己回收吗？`,该问题应该是属于操作系统相关的，于是我们在OperatingSystem这个目录项下面加上这个问题，然后再OperatingSystem这个文件夹下创建该问题的md文件就好。


## Overview

### Compiler
1. [为什么目标文件中未初始化的全局/静态变量要使用COMMON块？](./Compiler/q1.md)
2. [为什么静态运行库里面一个目标文件只包含一个函数？](./Compiler/q2.md)
   
   

### OperatingSystem
1. [为什么进程退出的时候没有内存泄漏？](./OperatingSystem/q1.md)
2. [分段真的是很糟粕的东西吗？]()
3. [为什么需要有memory allocator?]()
4. [为什么main“不是”程序的入口？](./OperatingSystem/q4.md)
5. [为什么需要多级页表，单级页表为何不妥？]()
6. [为什么有伙伴系统分配器还需要SLAB分配器？](./OperatingSystem/q6.md)
7. [为什么说mmap能共享内存？]()
8. [为什么不用fork来创建线程？]()
9. [为什么说Linux没有线程的概念？](./)

### Database

### Networking


### Architecture
1. [为什么C++里面的浮点数比较要用1e-6作基准？](./Architecture/q1.md)
2. 










