## C++并发编程实战一:基础使用、Join、传参、线程所有权转移 

参考资料:

[C++并发编程](https://www.zhihu.com/column/c_1307735602880331776)

[xiaoweiChen/CPP-Concurrency-In-Action-2ed-2019](https://github.com/xiaoweiChen/CPP-Concurrency-In-Action-2ed-2019/releases)

- 头文件: `#include <thread>`

- 最简单的线程启动:

  ```cpp
  # 启动一:
  void do_some_work();
  std::thread my_thread(do_some_work);
  ```

  ```cpp
  # 启动二:
  # 重载了()操作符的类
  class background_task
  {
  public:
   void operator()() const
   {
     do_something();
     do_something_else();
   }
  };
   
  background_task f;
  std::thread my_thread(f);
  ```

- 启动线程后，我们需要明确是等待线程结束(join) 还是 让它自主运行(detach)；

- **如果`std::thread`对象销毁前还没确定是 join 还是 detach，那么线程就会终止。`std::thread`的析构函数会调用`std::terminate()`**。

- 如果我们需要detach线程，那么我们**<mark style="color:red;">必须确保线程访问的数据直到线程结束之前都是有效的。</mark>**

  下面是线程函数访问局部变量指针 或 引用，并在函数退出后线程还没结束的反例

  ```cpp
  struct func
  {
    int& i;
    func(int& i_) : i(i_) {}
    void operator() ()
    {
      for (unsigned j=0 ; j<1000000 ; ++j)
      {
        do_something(i);           // 1 潜在访问隐患：悬空引用
      }
    }
  };
  void oops()
  {
    int some_local_state=0;
    func my_func(some_local_state);
    std::thread my_thread(my_func);
    my_thread.detach();          // 2 不等待线程结束
  }                              // 3 新线程可能还在运行
  ```

- 在函数内创建线程，并且需要访问函数的局部变量，那么尽量通过join确保线程在函数退出前完成；

- 调用`join()`,会清理线程相关的存储,这样`std::thread`对象将不再与已经结束的线程有任何关联。这意味着，**对一个线程只能调用一次`join()`**；一旦已经调用过`join()`，`std::thread`对象就不能再次连接，同时 `joinable()`将返回`false`。

- **如何确保异常场景下 线程的正确join?**

  方法一：异常分支逻辑、正常分支逻辑都调用join。(确保覆盖所有可能的路径非常痛苦)

  ```cpp
  struct func; // 定义在清单2.1中
  void f()
  {
    int some_local_state=0;
    func my_func(some_local_state);
    std::thread t(my_func);
    try
    {
      do_something_in_current_thread();
    }
    catch(...)
    {
      t.join();  // 1
      throw;
    }
    t.join();  // 2
  }
  ```

  方法二: **<mark style="color:red;">“资源获取即初始化”(RAII, Resource Acquisition Is Initialization)，并且提供一个类，在析构函数中`join`</mark>**

  ```cpp
  class thread_guard
  {
    std::thread& t;
  public:
    explicit thread_guard(std::thread& t_):
      t(t_)
    {}
    ~thread_guard()
    {
      if(t.joinable()) // 1
      {
        t.join();      // 2
      }
    }
    thread_guard(thread_guard const&)=delete;   // 3
    thread_guard& operator=(thread_guard const&)=delete;
  };
  struct func; // 定义在清单2.1中
  void f()
  {
    int some_local_state=0;
    func my_func(some_local_state);
    std::thread t(my_func);
    thread_guard g(t);
    do_something_in_current_thread();
  }    // 4
  ```

  - 函数f()返回，局部对象逆序销毁。`thread_guard`对象g是第一个被销毁的，此时线程`std::thread &t`在析构函数中被join。即使`do_something_in_current_thread()`抛出异常，这个销毁依然会发生。
  - 注意`t.joinable()`很重要；
  - 拷贝构造函数和拷贝赋值操作被标记为=delete，是为了不让编译器自动生成它们;

- 在后台运行的线程

  调用`detach`后让线程在后台运行，此时再也没直接的方法与其通信了。也没法等待线程结束。

  detach的线程经常叫做守护线程(daemon threads)。

### 参数传递

```cpp
# 字符串字面值作为char const *类型传递，新线程的上下文将其转换为一个std::string对象
void f(int i, std::string const& s);
std::thread t(f, 3, "hello");
```

- **错误案例一**

```cpp
#buffer是一个指针变量，指向局部变量，然后传递到新线程中。函数 oops退出后，buffer就被释放了，导致未定义行为
void f(int i,std::string const& s);
void oops(int some_param)
{
  char buffer[1024];
  sprintf(buffer, "%i",some_param);
  std::thread t(f,3,buffer);
  t.detach();
}
```

解决方案:

```cpp
void f(int i,std::string const& s);
void not_oops(int some_param)
{
  char buffer[1024];
  sprintf(buffer,"%i",some_param);
  std::thread t(f,3,std::string(buffer));  // 使用std::string，避免悬空指针
  t.detach();
}
```

- 错误案例二: **<mark style="color:red;">传递给线程构造函数的参数，会被多拷贝一次。目标函数接收引用时特别注意</mark>**

  ```cpp
  void update_data_for_widget(widget_id w,widget_data &data); //1
  void oops_again(widget_id w)
  {
      widget_data data;
      std::thread t(update_data_for_widget,w,data); // 2
      display_status();
      t.join();
      process_widget_data(data); // 3
  }
  ```

  - 虽然`update_data_for_widget`(1)的第二个参数期待传入一个引用，但是`std::thread`的构造函数(2)并不知晓;

    <mark style="color:red">`std::thread`的构造函数无视`update_data_for_widget`函数期待的参数类型，并盲目的拷贝已提供的变量data</mark>;

  - 为了照顾那些只能进行移动的类型，**这里是以右值的方式进行拷贝传递**；

  - 而后以右值为实参，调用 `update_data_for_widget()`函数;

  - 而`update_data_for_widget()`期待的是一个非常量引用作为参数，而非 右值作为参数，所以编译出错**。

    如果`update_data_for_widget()`期待的是一个常量引用`widget_data const &data`。那就不会编译出错。

    不过`process_widget_data(data)`得到的依然是旧数据；

  - 对于`std::bind`的开发者来说，问题的解决方法显而易见：可以使用`std::ref`将参数人为转换为引用的形式，从而将线程的调用改为以下形式:

    ```cpp
    std::thread t(update_data_for_widget,w,std::ref(data));
    ```

    **此时`std::thread`构造函数拷贝的是一个data的引用**，`update_data_for_widget`接收到的也是`data`变量的引用，而非一个`data`变量拷贝的引用。

- **<mark style="color:red;">线程中执行 类成员函数</mark>**

  ```cpp
  class X
  { 
      public:
      void do_lengthy_work();
  };
  X my_x;
  std::thread t(&X::do_lengthy_work,&my_x); // 1
  ```

  上面这段代码中，新线程将`my_x.do_lengthmy_work()`作为线程函数;

  `my_x`的地址(1)作为指针对象提供给函数。

  **线程中如何为类的成员函数 提供参数呢？**

  其实很简单，`std::thread`构造函数的第三个参数就是类成员函数的第一个参数，以此类推。

  ```cpp
  class X
  { 
  public:
     void do_lengthy_work(int);
  };
  X my_x;
  int num(0);
  std::thread t(&X::do_lengthy_work, &my_x, num);
  ```

  上面这段代码中，新线程将`my_x.do_lengthy_work(num)` 作为线程函数。

- **<mark style="color:red;">有一种情况，提供的线程的参数只能移动，不能拷贝。如`std::unique_ptr`</mark>**

  同一时间内，只允许一个`std::unique_ptr`实例指向一个给定对象，并且当这个实例销毁时，指向的对象也将被删除。

  当原对象是一个临时值时，移动自动完成，但当原对象是一个命名变量，那么需要调用`std::move()`来请求转移。

  ```cpp
  void process_big_object(std::unique_ptr<big_object>);
  std::unique_ptr<big_object> p(new big_object);
  p->prepare_data(42);
  std::thread t(process_big_object,std::move(p));
  
  #后续不能再直接调用 p;
  ```

### 转移线程所有权

想想两个场景:

a. 函数A，函数A中会生成一个线程在后台运行，线程后台运行后，函数A将线程的所有权返回给 自己的调用者；

b. 或者我们创建一个线程，然后将线程所有权 传递给 一个函数，由这个函数来等待线程结束(对线程进行join)。

两种场景中，我们都需要将 线程的所有权从一个地方转移到另一个地方。

C++中有很多资源占有(resource-owning)类型，如`std::ifstream`、`std::unique_ptr`、`std::thread`等，都是只可移动，不可拷贝的。所以`std::thread`的所有权在函数中移动，需要用`std::move()`。

```cpp
void some_function();
void some_other_function();
std::thread t1(some_function);            // 1
std::thread t2=std::move(t1);            // 2
t1=std::thread(some_other_function);    // 3
std::thread t3;                            // 4
t3=std::move(t2);                        // 5
t1=std::move(t3);                        // 6 赋值操作将使程序崩溃
```

- (1) 一个新线程启动并与t1关联;

- (2) t2获得 t1线程的所有权 ，变量t1和线程已经没任何关联，此时变量t2 拥有执行`some_function()`函数的线程的所有权;

- (3) 一个临时`std::thread`对象相关的线程启动了。随后所有权转移到t1。无需调用`std::move`;

- (4) t3是默认构造的，与任何执行线程都没有关联;

- (5) 显式调用`std::move()`将与t2关联线程的所有权转移到t3中。因为t2是一个命名对象。这些移动操作完成后:

  t1与执行`some_other_function`的线程相关联;

  t2没有关联任何线程;

  t3与执行`some_function`的线程相关联;

- (6) 最后一个移动操作，将`some_function`线程的所有权转移给t1;

  **t1已经有了一个关联的线程(运行`some_other_function`的线程)，所以这里系统直接调用`std::terminate()`终止程序运行**。

**线程所有权向函数外转移**

```cpp
std::thread f()
{
  void some_function();
  return std::thread(some_function);
}
std::thread g()
{
  void some_other_function(int);
  std::thread t(some_other_function,42);
  return t;
}
```

**线程所有权转移到函数中。函数接受一个值传递的`std::thread`实例即可**

```cpp
void f(std::thread t);
void g()
{
  void some_function();
  f(std::thread(some_function));
  std::thread t(some_function);
  f(std::move(t));
}
```

**`std::thread`支持移动，如果容器是移动感知(move-aware)的(比如最新的`std::vector<>`)，也允许`std::thread`对象的容器**

下面代码批量创建threads并等待他们结束:

```cpp
void do_work(unsigned id);
void f()
{
  std::vector<std::thread> threads;
  for(unsigned i=0; i < 20; ++i)
  {
    threads.push_back(std::thread(do_work,i)); // 产生线程
  } 
  for (auto &entry:threads) //每个线程调用join()
    entry.join();
}
```

### 运行时选择线程数量

- `std::thread::hardware_concurrency()` 返回CPU核数，当系统信息无法获取时，函数返回0；

- 下面是一个并发求和的示例:

  ```cpp
  template <typename Iterator, typename T>
  struct accumulate_block01
  {
  	void operator()(Iterator first, Iterator last, T &result)
  	{
  		result = std::accumulate(first, last, result);
  	}
  };
  
  template <typename Iterator, typename T>
  T parallel_accumulate001(Iterator first, Iterator last, T init)
  {
  	unsigned long long length = std::distance(first, last);
  	if (length == 0)
  	{
  		return init;
  	}
  	unsigned long const min_per_thread = 5;
  	unsigned long const max_threads = (length + min_per_thread - 1) / min_per_thread;
  	unsigned long const hardware_threads = std::thread::hardware_concurrency();
  	unsigned long const num_threads = std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);
  	unsigned long const block_size = length / num_threads;
  
  	std::vector<T> results(num_threads);
  	std::vector<std::thread> threads(num_threads - 1); //作为main 线程自己也算一个线程
  
  	Iterator block_start = first;
  	for (unsigned long i = 0; i < num_threads - 1; i++)
  	{
  		Iterator block_end = block_start;
  		std::advance(block_end, i * block_size);
  		threads[i] = std::thread(
  			accumulate_block01<Iterator, T>(),
  			block_start, block_end,
  			std::ref(results[i]));
  		block_start = block_end;
  	}
  	accumulate_block01<Iterator, T>()(block_start, last, results[num_threads - 1]); //main函数也计算一部分
  	for (auto &entry : threads)
  		entry.join();
  
  	return std::accumulate(results.begin(), results.end(), init);
  }
  
  int main()
  {
      std::vector<int> vi;
      for(int i=0;i<10;++i)
      {
          vi.push_back(10);
      }
      int sum=parallel_accumulate001(vi.begin(),vi.end(),5);
      std::cout<<"sum="<<sum<<std::endl;
  }
  ```

#### 标识线程

- 线程标识类型为:`std::thread::id`;
- 通过两种方式检索:
  - 第一种, 通过调用`std::thread`对象的成员函数`get_id()`来获取。如果`std::thread`对象没有与任何执行线程关联，`get_id()`将返回默认构造的`std::thread::id`对象。标识没有任何线程(not any thread)；
  - 第二种,调用`std::this_thread::get_id()`也可以获得当前线程的标识，该函数在`<thread>`头文件中;
- `std::thread::id`类型的对象**可随意拷贝 与 比较**。如果两个`std::thread::id`类型的对象相等，则代表同一个线程；
  - `std::thread::id`类型 可以做 容器的键、可以用于排序 等等。

