##问题
该如何理解下面的话：
既然每个线程至多有一个EventLoop对象，那么我们让EventLoop的static成员函数getEventLoopOfCurrentThread()返回这个对象。返回值可能为NULL，如果当前线程不是IO线程的话

# Muduo `getEventLoopOfCurrentThread()` 理解说明

## 原话

> 既然每个线程至多有一个 `EventLoop` 对象，那么我们让 `EventLoop` 的 `static` 成员函数 `getEventLoopOfCurrentThread()` 返回这个对象。返回值可能为 `NULL`，如果当前线程不是 IO 线程的话。

## 先说结论

这句话的意思是：

- `muduo` 约定一个线程里最多只有一个 `EventLoop`
- 所以可以给每个线程保存一个“当前线程对应的 `EventLoop*`”
- `getEventLoopOfCurrentThread()` 的作用，就是把这个线程对应的指针取出来
- 如果当前线程根本没有创建过 `EventLoop`，那这个指针就是 `NULL`

更白话一点：

**它相当于在问：“当前这个线程，有没有属于自己的 `EventLoop`？有就返回，没有就返回 `NULL`。”**

## 为什么能这么设计

因为 `muduo` 的核心约定就是：

- `one loop per thread`
- 一个线程最多一个 `EventLoop`

这在源码里也写得很清楚：

```cpp
/// Reactor, at most one per thread.
```

也就是说：

- 一个线程不会同时拥有两个 `EventLoop`
- 所以“当前线程对应哪个 `EventLoop`”这个问题，答案最多只有一个

既然答案最多只有一个，那就很适合提供一个静态函数直接取。

## 它到底是怎么实现的

关键实现其实非常简单。

### 1. 每个线程都有一个线程局部指针

`EventLoop.cc` 里定义了这样一个变量：

```27:31:/home/ericyiyang/QQMail/mmfddatagateway/mmfd3rd/muduo/muduo/net/EventLoop.cc
namespace
{
__thread EventLoop* t_loopInThisThread = 0;
```

这里的 `__thread` 可以先理解成：

**每个线程各自有一份独立的 `t_loopInThisThread`。**

注意这点很关键：

- 线程 A 有自己的 `t_loopInThisThread`
- 线程 B 也有自己的 `t_loopInThisThread`
- 它们互相不干扰

所以这个变量不是“所有线程共享一个值”，而是“每个线程都有自己的槽位”。

### 2. 静态函数只是把这个线程局部指针返回出去

```59:62:/home/ericyiyang/QQMail/mmfddatagateway/mmfd3rd/muduo/muduo/net/EventLoop.cc
EventLoop* EventLoop::getEventLoopOfCurrentThread()
{
  return t_loopInThisThread;
}
```

所以 `getEventLoopOfCurrentThread()` 并没有做复杂逻辑。  
它本质上只是：

- 取出当前线程自己的 `t_loopInThisThread`
- 然后返回

## 这个指针是什么时候被设置进去的

当某个线程创建 `EventLoop` 时，构造函数里会做这件事：

```77:86:/home/ericyiyang/QQMail/mmfddatagateway/mmfd3rd/muduo/muduo/net/EventLoop.cc
LOG_DEBUG << "EventLoop created " << this << " in thread " << threadId_;
if (t_loopInThisThread)
{
  LOG_FATAL << "Another EventLoop " << t_loopInThisThread
            << " exists in this thread " << threadId_;
}
else
{
  t_loopInThisThread = this;
}
```

这段代码表达了两个重要意思：

### 1. 当前线程如果已经有 `EventLoop` 了，就报错

也就是说：

- `muduo` 明确禁止一个线程里再创建第二个 `EventLoop`

这正是“每个线程至多一个 `EventLoop`”的直接落实。

### 2. 如果当前线程还没有 `EventLoop`，就把 `this` 存进去

也就是：

- 当前线程第一次创建 `EventLoop`
- 就把这个对象地址记到当前线程自己的 `t_loopInThisThread`

这样以后只要在这个线程里调用：

```cpp
EventLoop::getEventLoopOfCurrentThread()
```

就能把它取回来。

## 为什么返回值可能是 `NULL`

因为不是每个线程都会创建 `EventLoop`。

如果某个线程从来没创建过 `EventLoop`，那么它的线程局部变量：

```cpp
__thread EventLoop* t_loopInThisThread = 0;
```

初始值就是 `0`，也就是 `NULL`。

所以：

- IO 线程通常会创建 `EventLoop`
- 普通线程如果没创建，就没有对应的 `EventLoop`
- 这时 `getEventLoopOfCurrentThread()` 返回 `NULL`

因此原话里说：

> 返回值可能为 `NULL`，如果当前线程不是 IO 线程的话

这个说法是合理的。

不过更严谨一点，可以改成：

**如果当前线程没有创建 `EventLoop`，那么返回值就可能是 `NULL`。**

因为“不是 IO 线程”只是常见情况，本质原因其实是：

- **这个线程没有 `EventLoop`**

## 为什么说“当前线程不是 IO 线程”时可能为 `NULL`

在 `muduo` 的常见使用模型里：

- main loop 所在线程是 IO 线程
- sub loop 所在线程也是 IO 线程
- 这些线程都会持有自己的 `EventLoop`

而有些普通工作线程、辅助线程、非网络线程：

- 可能并不跑事件循环
- 也不会创建 `EventLoop`

所以在这些线程里调用：

```cpp
EventLoop::getEventLoopOfCurrentThread()
```

通常就会得到 `NULL`。

## 析构时为什么又把它设回 `NULL`

`EventLoop` 析构函数里还有这一句：

```93:100:/home/ericyiyang/QQMail/mmfddatagateway/mmfd3rd/muduo/muduo/net/EventLoop.cc
EventLoop::~EventLoop()
{
  LOG_DEBUG << "EventLoop " << this << " of thread " << threadId_
            << " destructs in thread " << CurrentThread::tid();
  wakeupChannel_->disableAll();
  wakeupChannel_->remove();
  ::close(wakeupFd_);
  t_loopInThisThread = NULL;
}
```

意思是：

- 当前线程的 `EventLoop` 被销毁后
- 线程局部槽位也要清空

这样做的好处是：

- 避免留下悬空指针
- 表示“这个线程现在已经没有 `EventLoop` 了”

## 该如何形象理解

你可以把它想成：

- 每个线程都有一个自己的小抽屉
- 抽屉名字叫 `t_loopInThisThread`
- 如果这个线程创建了 `EventLoop`，就把指针放进抽屉
- `getEventLoopOfCurrentThread()` 就是去当前线程自己的抽屉里拿
- 抽屉里没东西，就返回 `NULL`

这个设计的关键优点是：

- 取当前线程的 `EventLoop` 非常快
- 不需要全局 map 查找
- 线程之间互不干扰

## 这句话容易误解的地方

### 误解 1：`static` 成员函数是不是意味着全局共享一个 `EventLoop`

不是。

这里 `static` 的意思只是：

- 这个函数属于类本身
- 调用它不需要某个具体对象

真正区分线程的是：

- 它返回的内容来自 `__thread EventLoop* t_loopInThisThread`

所以虽然函数是类级别的，但返回值仍然是“当前线程自己的那一份”。

### 误解 2：返回 `NULL` 是不是说明出错了

不一定。

返回 `NULL` 只说明：

- 当前线程没有属于自己的 `EventLoop`

这有可能是正常情况。

### 误解 3：是不是所有线程都应该有 `EventLoop`

不是。

只有那些需要跑事件循环的线程，通常才会创建 `EventLoop`。

## 最终结论

可以把原话总结成下面这段：

> `muduo` 假设一个线程最多只有一个 `EventLoop`，因此它用线程局部变量 `t_loopInThisThread` 记录“当前线程对应的 EventLoop 指针”。`getEventLoopOfCurrentThread()` 本质上就是返回这个线程局部指针。如果当前线程从未创建过 `EventLoop`，那么这个值就是 `NULL`。所以它的作用，就是快速获取“当前线程自己的 EventLoop”。`

## 一句话版

**`getEventLoopOfCurrentThread()` 不是去“全局找 EventLoop”，而是去“当前线程自己的线程局部存储里取 EventLoop 指针”；有就返回，没有就返回 `NULL`。**
