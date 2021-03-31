# WhyThis
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






OS
=====

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
## Networking


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
* 
