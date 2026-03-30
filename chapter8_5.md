# 问题
事件循环必须在IO线程执行，因此EventLoop：​：loop()会检查这一pre-condi-tion(L44)。本节的loop()什么事都不做，等5秒就退出。
# muduo EventLoop 代码实战分析

> 本文用通俗语言逐行分析 EventLoop::loop() 的实现，以及两个测试用例（正面测试 test1.cc 和负面测试 test2.cc），帮助理解 muduo 的"线程归属检查"是如何在实际代码中工作的。

---

## 一、EventLoop::loop() 逐行分析

### 1.1 源代码（EventLoop.cc）

```cpp
void EventLoop::loop()
{
    assert(!looping_);          // L43
    assertInLoopThread();       // L44
    looping_ = true;            // L45

    ::poll(NULL, 0, 5*1000);    // L47

    LOG_TRACE << "EventLoop " << this << " stop looping";  // L49
    looping_ = false;           // L50
}
```

### 1.2 逐行解释

| 行号 | 代码 | 通俗解释 |
|------|------|----------|
| L43 | `assert(!looping_)` | **防止重复启动。** `looping_` 是一个标记，表示"事件循环是否正在运行"。如果已经在运行了（`looping_` 为 true），这里就会触发断言失败，程序终止。类比：一台洗衣机正在转，你不能再按一次"开始"。 |
| L44 | `assertInLoopThread()` | **检查线程归属。** 确保调用 `loop()` 的线程，就是创建这个 EventLoop 的那个线程。如果不是，程序立刻崩溃。类比：只有这个灶台的负责厨师才能开火，别人不能碰。 |
| L45 | `looping_ = true` | 标记为"正在运行"。通过了前面两项检查后，正式开始事件循环。 |
| L47 | `::poll(NULL, 0, 5*1000)` | **当前版本只是"假装干活"。** `poll` 是一个系统调用，这里传入的参数意思是"什么事件都不监听，等 5 秒后返回"。后续版本会替换成真正的事件监听逻辑。类比：厨师进了厨房，但今天没有订单，就站着等了 5 秒钟然后下班。 |
| L49 | `LOG_TRACE << ...` | 打印一条日志，表示事件循环结束了。 |
| L50 | `looping_ = false` | 标记为"已停止运行"，重置状态。 |

### 1.3 关键理解

原文说的"事件循环必须在 IO 线程执行"，对应的就是 **L44 这一行**。它调用了 `assertInLoopThread()`，确保 `loop()` 只能被创建 EventLoop 的那个线程调用。如果用错了线程，程序会在这一行直接崩溃。

---

## 二、test1.cc 正面测试（正确用法）

### 2.1 源代码

```cpp
void threadFunc()
{
    printf("threadFunc(): pid = %d, tid = %d\n",
           getpid(), muduo::CurrentThread::tid());

    muduo::EventLoop loop;     // 在子线程中创建 EventLoop
    loop.loop();               // 在子线程中调用 loop()
}

int main()
{
    printf("main(): pid = %d, tid = %d\n",
           getpid(), muduo::CurrentThread::tid());

    muduo::EventLoop loop;     // 在主线程中创建 EventLoop

    muduo::Thread thread(threadFunc);
    thread.start();            // 启动子线程

    loop.loop();               // 在主线程中调用 loop()
    pthread_exit(NULL);
}
```

### 2.2 通俗解释

这个程序有**两个线程**，每个线程做的事情可以用下面的图来理解：

```
主线程（main）                      子线程（threadFunc）
    │                                    │
    ▼                                    ▼
创建 EventLoop A                    创建 EventLoop B
（记录：A 属于主线程）                （记录：B 属于子线程）
    │                                    │
    ▼                                    ▼
调用 A.loop()                       调用 B.loop()
    │                                    │
    ▼                                    ▼
assertInLoopThread()                assertInLoopThread()
检查：当前是主线程，                  检查：当前是子线程，
      A 属于主线程                         B 属于子线程
      ──→ 通过!                           ──→ 通过!
    │                                    │
    ▼                                    ▼
等待 5 秒后退出                      等待 5 秒后退出
```

**结果：程序正常运行，正常退出。**

### 2.3 为什么没问题？

关键在于：**每个 EventLoop 都在创建它的线程中调用 `loop()`**。

- EventLoop A 在主线程创建 → 在主线程调用 `loop()` → 线程 ID 匹配 → 通过
- EventLoop B 在子线程创建 → 在子线程调用 `loop()` → 线程 ID 匹配 → 通过

> 类比：厨师 A 管 A 灶台，厨师 B 管 B 灶台，各管各的，没有冲突。

---

## 三、test2.cc 负面测试（错误用法）

### 3.1 源代码

```cpp
muduo::EventLoop* g_loop;       // 全局指针

void threadFunc()
{
    g_loop->loop();              // 子线程试图调用主线程创建的 EventLoop
}

int main()
{
    muduo::EventLoop loop;       // 在主线程中创建 EventLoop
    g_loop = &loop;              // 把地址存到全局指针
    muduo::Thread t(threadFunc);
    t.start();                   // 启动子线程
    t.join();                    // 等待子线程结束
}
```

### 3.2 通俗解释

```
主线程（main）                      子线程（threadFunc）
    │
    ▼
创建 EventLoop
（记录：属于主线程）
    │
    ▼
把 EventLoop 的地址
存到全局指针 g_loop
    │
    ▼
启动子线程 ─────────────────────────→ 子线程开始执行
    │                                    │
    │                                    ▼
    │                               通过 g_loop 调用 loop()
    │                                    │
    │                                    ▼
    │                               assertInLoopThread() 检查：
    │                               当前是子线程，
    │                               但 EventLoop 属于主线程
    │                               ──→ 不匹配!
    │                                    │
    │                                    ▼
    │                               abortNotInLoopThread()
    │                               打印错误日志，程序崩溃终止!
    │                                    ✕
    ▼
等待子线程（但子线程已崩溃）
```

**结果：程序因为断言失败而异常终止。**

### 3.3 为什么会崩溃？

- EventLoop 是在**主线程**创建的，所以 `threadId_` 记录的是主线程的 ID
- 子线程通过全局指针 `g_loop` 调用了 `loop()`
- `loop()` 内部的 `assertInLoopThread()` 一检查：当前线程 ID（子线程）和 `threadId_`（主线程）**不一样**
- 检查不通过 → 调用 `abortNotInLoopThread()` → 程序崩溃

> 类比：厨师 A 的灶台，被服务员 B 偷偷去开火了。安全系统立刻检测到"操作人不是指定厨师"，直接拉响警报、关掉整个厨房。

### 3.4 这个测试的意义

test2.cc 是**故意写错的代码**，目的是验证 muduo 的防护机制是否有效。结果证明：muduo 确实能在运行时检测到"在错误的线程调用 EventLoop"，并及时终止程序，防止更严重的后果。

---

## 四、练习题

> 原文练习：写一个负面测试，在主线程创建两个 EventLoop 对象，验证程序会异常终止。

### 4.1 为什么会异常终止？

回忆之前说的规则：**一个线程只能运行一个 EventLoop**。

如果在主线程创建两个 EventLoop，那就违反了这个规则。muduo 在 EventLoop 的构造函数中就会检查：当前线程是否已经有一个 EventLoop 了？如果已经有了，第二个 EventLoop 创建时就会检测到冲突，程序终止。

### 4.2 参考代码思路

```cpp
int main()
{
    muduo::EventLoop loop1;   // 第一个 EventLoop，正常创建
    muduo::EventLoop loop2;   // 第二个 EventLoop，检测到当前线程已有 loop1
                              // → 断言失败，程序崩溃终止
}
```

> 类比：一个厨房只能有一个灶台负责人。如果已经指定了厨师 A，再指定一个厨师 B，管理系统就会报错："这个厨房已经有人管了！"

---

## 五、总结

| 知识点 | 一句话解释 |
|--------|-----------|
| `EventLoop::loop()` | 事件循环的入口，启动前会做两项安全检查 |
| `assert(!looping_)` | 防止同一个 EventLoop 被重复启动 |
| `assertInLoopThread()` | 防止在错误的线程中调用 loop() |
| test1.cc（正面测试） | 每个线程创建并使用自己的 EventLoop → 正常运行 |
| test2.cc（负面测试） | 子线程试图使用主线程的 EventLoop → 程序崩溃 |
| 练习（两个 EventLoop） | 同一线程创建两个 EventLoop → 违反"一线程一循环"规则，程序崩溃 |
| 核心原则 | **一个 EventLoop 只属于一个线程，一个线程只运行一个 EventLoop** |

**设计哲学：muduo 通过运行时断言检查，把"用错线程"这种隐蔽的 bug 变成了"立即崩溃"的显式错误，让问题在开发阶段就暴露出来，而不是上线后才出现诡异的故障。**
