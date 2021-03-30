# WhyThis
This repo will contain all my questions about system designing and my opinions about them. I'll try to update it everyday. :-)

## C++
* 为什么有了`NULL`，`C++11`又引入了`nullptr`?

----
专门用来区分0和空指针。

----

* 为什么`C++`要引入`constexpr`

----
为了解决C++标准中数组的长度必须是一个常量表达式的问题。

----


## OS
### 条件变量为什么需要加锁和循环

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

## Networking


## Compiler


## Database



