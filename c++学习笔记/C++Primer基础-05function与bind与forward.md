## C++Primer基础:std::function、std::bind、std::forward

原文: [搞定c++11新特性std::function和lambda表达式](https://mp.weixin.qq.com/s/6zzF8GEgpMsNrdoBLi5csA)

#### std::function

满足一下条件之一就可称为 **可调用对象**？

- 是一个函数指针;
- 是一个**具有`operator()`成员函数的类对象**;
- 是一个`lambda`表达式;
- 是一个 **可被转换为函数指针** 的类对象;
- 是一个**类成员函数指针**;
- bind表达式 或 其他函数对象;

`std::function`就是上面这个可调用对象的封装器，可以把`std::function`看做一个函数对象。`std::function`实例可以存储、复制赫尔调用任何可调用对象，存储的可调用对象称为`std::function`的目标，若`std::function`不含目标，则称它为空，调用空的`std::function`会抛出`std::bad_function_call`异常。

参考如下实例代码:

```cpp
#include <functional>
#include <iostream>

struct Foo {
   Foo(int num) : num_(num) {}
   void print_add(int i) const { std::cout << num_ + i << '\n'; }
   int num_;
};

void print_num(int i) { std::cout << i << '\n'; }

struct PrintNum {
   void operator()(int i) const { std::cout << i << '\n'; }
};

int main() {
   // 存储自由函数
   std::function<void(int)> f_display = print_num;
   f_display(-9);

   // 存储 lambda
   std::function<void()> f_display_42 = []() { print_num(42); };
   f_display_42();

   // 存储到 std::bind 调用的结果
   std::function<void()> f_display_31337 = std::bind(print_num, 31337);
   f_display_31337();

   // 存储到成员函数的调用
   std::function<void(const Foo&, int)> f_add_display = &Foo::print_add;
   const Foo foo(314159);
   f_add_display(foo, 1);
   f_add_display(314159, 1);

   // 存储到数据成员访问器的调用
   std::function<int(Foo const&)> f_num = &Foo::num_;
   std::cout << "num_: " << f_num(foo) << '\n';

   // 存储到成员函数及对象的调用
   using std::placeholders::_1;
   std::function<void(int)> f_add_display2 = std::bind(&Foo::print_add, foo, _1);
   f_add_display2(2);

   // 存储到成员函数和对象指针的调用
   std::function<void(int)> f_add_display3 = std::bind(&Foo::print_add, &foo, _1);
   f_add_display3(3);

   // 存储到函数对象的调用
   std::function<void(int)> f_display_obj = PrintNum();
   f_display_obj(18);
}
```

当给std::function填入合适的参数表和返回值后，它就变成了可以容纳所有这一类调用方式的函数封装器。

std::function还可以用作回调函数，或者在C++里如果需要使用回调那就一定要使用std::function，特别方便，这方面的使用方式大家可以读下我之前写的关于线程池和定时器相关的文章。

##### std::bind

使用std::bind可以将可调用对象和参数一起绑定，绑定后的结果使用std::function进行保存，并延迟调用到任何我们需要的时候。

std::bind通常有两大作用：

- **将可调用对象与参数一起绑定为另一个std::function供调用**;
- **将n元可调用对象转成m(m < n)元可调用对象，绑定一部分参数，这里需要使用std::placeholders**

示例:

```cpp
#include <functional>
#include <iostream>
#include <memory>

void f(int n1, int n2, int n3, const int& n4, int n5) {
   std::cout << n1 << ' ' << n2 << ' ' << n3 << ' ' << n4 << ' ' << n5 << std::endl;
}

int g(int n1) { return n1; }

struct Foo {
   void print_sum(int n1, int n2) { std::cout << n1 + n2 << std::endl; }
   int data = 10;
};

int main() {
   using namespace std::placeholders;  // 针对 _1, _2, _3...

   // 演示参数重排序和按引用传递
   int n = 7;
   // （ _1 与 _2 来自 std::placeholders ，并表示将来会传递给 f1 的参数）
   auto f1 = std::bind(f, _2, 42, _1, std::cref(n), n);
   n = 10;
   f1(1, 2, 1001);  // 1 为 _1 所绑定， 2 为 _2 所绑定，不使用 1001
                    // 进行到 f(2, 42, 1, n, 7) 的调用

   // 嵌套 bind 子表达式共享占位符
   auto f2 = std::bind(f, _3, std::bind(g, _3), _3, 4, 5);
   f2(10, 11, 12);  // 进行到 f(12, g(12), 12, 4, 5); 的调用

   // 绑定指向成员函数指针
   Foo foo;
   auto f3 = std::bind(&Foo::print_sum, &foo, 95, _1);
   f3(5); //foo->print_sum(95,5);

   // 绑定指向数据成员指针
   auto f4 = std::bind(&Foo::data, _1);
   std::cout << f4(foo) << std::endl;

   // 智能指针亦能用于调用被引用对象的成员
   std::cout << f4(std::make_shared<Foo>(foo)) << std::endl;
}
```

#### std::forward

`std::forward`通常是用于完美转发的，它会将输入的参数原封不动地传递到下一个函数中，这个“原封不动”指的是，如果输入的参数是左值，那么传递给下一个函数的参数的也是左值；如果输入的参数是右值，那么传递给下一个函数的参数的也是右值。一个经典的完美转发的场景是:

```cpp
template <typename... Args>
void forward(Args&&... args) {
    f(std::forward<Args>(args)...);
}
```

1. 由于我们需要保留输入参数的右值属性, 因此`Args`后面需要跟上`&&`;
2. `std::forward`的模板参数必须是`<Args>`，而不能是`<Args...>`，这是由于我们不能对Args进行解包之后传递给`std::forward`,而解包的过程必须在调用std::forward之后;

下面是线程池中提交任务到线程池的一段代码

```cpp
template <typename F, typename... Args>
auto submit(F &&f, Args &&...args) -> std::future<decltype(f(args...))>
{
    // Create a function with bounded parameters ready to execute
    std::function<decltype(f(args...))()> func = std::bind(std::forward<F>(f), std::forward<Args>(args)...);
    
    // 异步执行,获取结果的基础
    auto task_ptr = std::make_shared<std::packaged_task<decltype(f(args...))()>>(func);
   
    //将packaged task 封装到一个void 函数中
    std::function<void()> wrapper_func = [task_ptr]()
    {
        (*task_ptr)();
    };
    //函数压入安全队列
    m_queue.push(wrapper_func);

    //唤醒一个等待中的线程
    m_conditional_lock.notify_one();

    //返回先前注册的任务的future指针
    return task_ptr->get_future();
}
```

