"类是不可拷贝的" 如何理解
这句话的意思是：该类的对象不能通过拷贝构造函数或拷贝赋值运算符来创建副本。

1. 什么是拷贝？
在 C++ 中，拷贝一个对象有两种方式：

MyClass a;
MyClass b(a);      // 拷贝构造
MyClass c = a;     // 拷贝构造（等价写法）
MyClass d;
d = a;             // 拷贝赋值 
2. 如何让类不可拷贝？
C++11 及以后：用 = delete 显式禁用拷贝构造和拷贝赋值：

class NonCopyable {
public:
    NonCopyable() = default;

    // 禁止拷贝
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;

    // 通常仍允许移动
    NonCopyable(NonCopyable&&) = default;
    NonCopyable& operator=(NonCopyable&&) = default;
}; 

4. 直观理解
你可以这样类比理解：

可拷贝的类 → 像一份文档，可以随便复印
不可拷贝的类 → 像一把钥匙，对应唯一一把锁，复印没有意义（或会造成安全问题）

5. 不可拷贝 ≠ 不可移动
不可拷贝的类通常仍然可以移动（move）。移动是"资源转移"，而拷贝是"资源复制"：

NonCopyable a;
// NonCopyable b = a;           // ❌ 编译错误：拷贝被禁止
NonCopyable c = std::move(a);   // ✅ 移动是允许的（资源从 a 转移到 c） 
比如 std::unique_ptr 就是典型的不可拷贝但可移动的类——你不能让两个 unique_ptr 同时拥有一块内存，但可以把所有权从一个转移给另一个。
