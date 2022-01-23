## C++并发编程实战: mutex、lock、unique_lock、call_once、shared_lock

#### 使用互斥锁`std::mutex`

通过构建一个`std::mutex`实例创建互斥锁，通过成员函数`lock`对互斥锁上锁，`unlock`进行解锁。

**C++标准库为互斥锁提供了一个RAII语法`std::lock_guard`类模块，构造时提供lock住互斥锁，析构时进行unlock**。

```cpp
#include <list>
#include <mutex>
#include <algorithm>
 
std::list<int> some_list;    // 1
std::mutex some_mutex;    // 2
 
void add_to_list(int new_value)
{
 std::lock_guard<std::mutex> guard(some_mutex);    // 3
 some_list.push_back(new_value);
}
 
bool list_contains(int value_to_find)
{
 std::lock_guard<std::mutex> guard(some_mutex);    // 4
 return std::find(some_list.begin(),some_list.end(),value_to_find) != some_list.end();
}
```

#### 死锁

死锁的解决方式 一般是 让两个互斥锁总是以相同的顺序上锁来避免。

不过我们考虑一种场景： 一个操作对同一个类的两个不同实例进行数据的swap，为了保证数据被正确的swap，就要避免数据被并发修改，每个实例上的互斥锁都必须锁住。但是，如果选择一个固定的顺序(例如，先是第一个参数对应的互斥锁，然后是第二个参数的互斥锁)，这可能会适得其反：只要交换一下参数的位置，两个线程试图在相同的两个实例间交换数据，就会发生死锁！

**<mark style="color:red;">C++标准库方法: `std::lock`，该函数可以一次性锁住两个 or 更多的互斥锁，且没有死锁的风险</mark>**

```cpp
// 这里的std::lock()需要包含<mutex>头文件
class some_big_object;
void swap(some_big_object& lhs,some_big_object& rhs);
class X
{
private:
  some_big_object some_detail;
  std::mutex m;
public:
  X(some_big_object const& sd):some_detail(sd){}
  friend void swap(X& lhs, X& rhs)
  {
    if(&lhs==&rhs)
      return;
    std::lock(lhs.m,rhs.m); // 1
    std::lock_guard<std::mutex> lock_a(lhs.m,std::adopt_lock); // 2
    std::lock_guard<std::mutex> lock_b(rhs.m,std::adopt_lock); // 3
    swap(lhs.some_detail,rhs.some_detail);
  }
};
```

- 首先检查参数是不是不同的实例；
- 调用`std::lock`锁住两个互斥锁;
- 创建两个`std::lock_guard`实例，额外提供的`std::adopt_lock`参数告诉`std::lock_guard`互斥锁已经上锁了，接过互斥锁的所有权即可，不用在`std::lock_guard`的构造函数里上锁。
- 使用`std::lock`去锁住`lhs.m`或`rhs.m`时，可能会抛出异常；这种情况下，异常会传播到`std::lock`之外;
- 当`std::lock`成功的获取了一个互斥锁上的锁，并且当其尝试从另一个互斥锁上再获取锁时，如果有异常抛出，第一个锁会自动释放:`std::lock`给互斥锁上锁的时候提供了要么全做要么不做的语义(all-or-nonthing)

C++17对上面这种情况提供流量支持，`std::scoped_lock`，**它接受一个互斥锁类型的列表作为模板参数**，功能上等同于`std::lock`+`std::lock_guard`。如:

```cpp
void swap(X& lhs, X& rhs)
{
 if(&lhs==&rhs)
 return;
 std::scoped_lock guard(lhs.m,rhs.m); // 1
 swap(lhs.some_detail,rhs.some_detail);
}
```

C++17另一个特性：类模板参数自动推导。上面`std::scoped_lock`就自动推导了参数类型。

```cpp
std::scoped_lock guard(lhs.m,rhs.m);
等同于
std::scoped_lock<std::mutex,std::mutex> guard(lhs.m,rhs.m);
```

#### 人为主动释放锁(控制锁粒度):`std::unique_lock`

比如某个线程持有锁，而后去执行文件I/O，将极大降低并发性能。

```cpp
void get_and_process_data()
{
  std::unique_lock<std::mutex> my_lock(the_mutex);
  some_class data_to_process=get_next_data_chunk();
  my_lock.unlock();  // 1 不要让锁住的互斥量越过process()函数的调用
  result_type result=process(data_to_process);
  my_lock.lock(); // 2 为了写入数据，对互斥量再次上锁
  write_result(data_to_process,result);
}
```

### 保护共享数据

#### 初始化期间保护数据:`std::once_flag`和`std::call_once`

数据初始化时需要写，初始化完成后，只需读即可。

```cpp
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag;  // 1
void init_resource()
{
  resource_ptr.reset(new some_resource);
}
void foo()
{
  std::call_once(resource_flag,init_resource);  // 可以完整的进行一次初始化
  resource_ptr->do_something();
}
```

再比如在class中:

```cpp
class X
{
private:
  connection_info connection_details;
  connection_handle connection;
  std::once_flag connection_init_flag;
  void open_connection()
  {
    connection=connection_manager.open(connection_details);
  }
public:
  X(connection_info const& connection_details_):
      connection_details(connection_details_)
  {}
  void send_data(data_packet const& data)  // 1
  {
    std::call_once(connection_init_flag,&X::open_connection,this);  // 2
    connection.send_data(data);
  }
  data_packet receive_data()  // 3
  {
    std::call_once(connection_init_flag,&X::open_connection,this);  // 2
    return connection.receive_data();
  }
};
```

第一次调用`send_data()`或`receive_data()`的线程完成初始化过程。

使用成员函数`open_connection()`完成初始化数据；

和其他标准库一样，`std::call_once()`调用`open_connection()`函数，需要传入this参数。

注意: `std::mutex`和`std::once_flag`是不能拷贝和移动的，需要显示定义相关成员函数，对这类成员进行操作。

#### 包含不常更新的数据结构: 读写锁,`std::shared_mutex`

如DNS表，更新很少，大部分是读取。但是更新是有的。

此时使用`std::mutex`来保护数据结构，显得非常反应过度。此时我们更需要一种"读-写锁"。

C++17标准库提供的互斥量:`std::shared_mutex`和`std::shared_timed_mutex`;

C++14标准库提供的互斥量`std::shard_timed_mutex`。

对于更新操作，使用`std::lock_guard<std::shared_mutex>`和`std::unique_lock<std::shard_mutex>`上锁；

对于读取操作，使用`std::shared_lock<std::shared_mutex>`获取访问权限。

```cpp
#include <map>
#include <string>
#include <mutex>
#include <shared_mutex>
class dns_entry;
class dns_cache
{
  std::map<std::string,dns_entry> entries;
  mutable std::shared_mutex entry_mutex;
public:
  dns_entry find_entry(std::string const& domain) const
  {
    std::shared_lock<std::shared_mutex> lk(entry_mutex);  // 1
    std::map<std::string,dns_entry>::const_iterator const it=
       entries.find(domain);
    return (it==entries.end())?dns_entry():it->second;
  }
  void update_or_add_entry(std::string const& domain,
                           dns_entry const& dns_details)
  {
    std::lock_guard<std::shared_mutex> lk(entry_mutex);  // 2
    entries[domain]=dns_details;
  }
};
```

### 嵌套锁:`std::recursive_mutex`

当一个线程已经获取一个`std::mutex`时(已经上锁)，并对其再次上锁，这个操作就是错误的，并且继续尝试这样做的话，就会产生未定义行为。然而，在某些情况下，一个线程尝试获取同一个互斥量多次，而没有对其进行一次释放是可以的。

互斥量锁住其他线程前，必须释放拥有的所有锁，所以当调用lock()三次后，也必须调用unlock()三次。

正确使用`std::lock_guard<std::recursive_mutex>`和`std::unique_lock<std::recursive_mutex>`可以帮你处理这些问题。