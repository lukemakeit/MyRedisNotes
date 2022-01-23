## C++ Primer 标准库: 动态内存

- 静态内存：保存局部`static`对象。类`static`数据成员以及定义在任何函数之外的变量。使用时分配，在程序结束时销毁；
- 栈内存: 用来保存定义在函数内的非`static`对象。数据块中有效，数据块结束后，回收内存；
- 堆内存：程序用堆来存储**动态分配**的对象；动态对象的生存周期由 程序来控制。

#### 动态内存与智能指针
- `new`和`delete`;
- `shared_ptr` 允许多个指针指向同一个对象；
- `unique_ptr` 则"独占"所指向的对象；
- `weak_ptr` 表示一种弱引用；
- 头文件`memory`;

#### `shared_ptr`类
```cpp
shared_ptr<string> p1; // shared_ptr,可以指向 string
shared_ptr<list<int>> p2; // shard_ptr, 可以指向 int 的list
```
默认初始化的智能指针中保存这一个空指针。
智能指针的使用方式与普通指针类似。解引用一个智能指针返回它指向的对象。如果在一个条件判断中使用智能指针，效果就是检测它是否为空:

```cpp
// if p1 is not null, check whether it's the empty string
if (p1 && p1->empty())
    *p1 = "hi"; // if so, dereference p1 to assign a new value to that string
```
![d7673a6fa438ac581882eaf774fd4d1a](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/C660748C-F1EF-4F9C-BB4F-5CD9CAB59026.png)
![c026b171168fcf31deb0f6bb612f6f6c](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/5045F39C-EB78-4F33-940A-67C221BD1933.png)

**make_shared函数**
最安全的分配和试用动态内存的方法是调用一个名为`make_shared`的标准库函数。
```cpp
// 指向一个值为42的int的shard_ptr
shared_ptr<int> p3 = make_shared<int>(42);
// p4 指向一个值为"9999999999"的string
shared_ptr<string> p4 = make_shared<string>(10, '9');
// p5 指向一个值初始化的int，即，值为0
shared_ptr<int> p5 = make_shared<int>();
```
当然我们通常使用`auto`定义一个对象来保存`make_shared`的结果，这种方式比较简单:
```cpp
// p6 指向一个动态分配的空 vector<string>
auto p6 =  make_shard<vector<string>>();
```

**`shared_ptr`的拷贝和赋值**
当进行拷贝和赋值操作时，每个`shared_ptr`都会记录有多少个其他的`shared_ptr`指向相同的对象:

```cpp
auto p = make_shared<int>(42); // p指向的对象只有一个p一个引用者
auto q(p); // p和q指向相同对象，此对象有两个引用者 
```
我们可以认为每个`shared_ptr`都有一个关联的计数器，通常称为 **引用计数(reference count)**。
无论何时我们拷贝一个`shared_ptr`，计数器都会增加。
当我们给`shared_ptr`的赋予一个新值 或是 `shared_ptr`被销毁(例如局部的`shared_ptr`离开作用域)时，计数器引用个数就会递减。
一旦一个`shared_ptr`计数器变为0，它就会自动释放自己管理的对象。

**`shared_ptr`自动销毁所管理的对象......**
当对象的`shared_ptr`引用计数减少为0时，`shared_ptr`就会自动销毁此对象。它是通过另一个特殊的成员函数——析构函数来完成销毁工作的。

`shared_ptr`的析构函数会递减它所指向的对象的引用计数。如果引用计数变为0，`shared_ptr`的析构函数就会销毁对象，并释放它们所占用的内存。

**`shared_ptr`还会自动释放相关联的内存**
```cpp
// factory 返回一个shared_ptr，指向一个动态分配的对象
shared_ptr<Foo> factory(T arg)
{
    // 恰当处理 arg
    // shared_ptr 负责释放内存
    return make_shared<Foo>(arg);
}

void use_factory(T arg)
{
    shared_ptr<Foo> p = factory(arg);
    // 使用 p
} // p离开了作用域，它指向的内存就会被自动释放掉
```

#### 直接管理内存
```cpp
string *ps = new string;
int *pi = new int;

int *pi = new int(1024); // object to which pi points has value 1024
string *ps = new string(10, '9'); // *ps is "9999999999"

// vector with ten elements with values from 0 to 9
vector<int> *pv = new vector<int>{0,1,2,3,4,5,6,7,8,9};

string *ps1 = new string; // default initialized to the empty string
string *ps = new string(); // value initialized to the empty string
int *pi1 = new int; // default initialized; *pi1 is undefined
int *pi2 = new int(); // value initialized to 0; *pi2 is 0
```

**释放动态内存**
```cpp
delete p;
```

**指针值和delete**
我们传递给`delete`的指针必须指向动态分配的内存，或者是一个空指针。释放一块并非`new`分配的内存，或者将相同的指针值释放多次。其行为是未定义的:
```cpp
int i, *pi1 = &i, *pi2 = nullptr;
double *pd = new double(33), *pd2 = pd;
delete i; // error: i 不是一个指针
delete pi1; // undefined: pi1 指向一个局部变量
delete pd; // ok
delete pd2; // undefined: 指向的内存已经被释放
delete pi2; // ok: 释放一个空指针总是没有错误的
```

**delete之后重置指针**
当我们delete一个指针后，指针值就变为无效。虽然指针无效了，但是在很多机器上指针任然保存着动态内存的地址。
在`delete`之后，指针就变成了人们所说的**悬空指针(dangling pointer)**，即，指向一块曾经保存数据对象但现在已经无效的内存的指针。
所以我们在`delete`之后，将`nullptr`赋予指针。
```cpp
int *p(new int(42)); // p 指向动态内存
auto q = p; // p and q 指向相同的内存
delete p; //  both p and q 均变为无效
p = nullptr; // that p 不再绑定任何对象
```

#### `shared_ptr`和`new`结合使用
```cpp
shared_ptr<int> p1 = new int(1024); // shared_ptr 可以指向一个double
shared_ptr<int> p2(new int(1024)); // p2 指向一个值为42的int
```
接收指针参数 的 智能指针 构造函数是`explicit`的。因此，我们不能将一个 内置指针隐式 转换为一个智能指针，必须使用直接初始化形式。
```cpp
shared_ptr<int> p1 = new int(1024); // error: 必须使用直接初始化形式
shared_ptr<int> p2(new int(1024)); // ok: 使用了直接初始化形式

shared_ptr<int> clone(int p) {
    return new int(p); // error: 隐式转换为shared_ptr<int>
}
shared_ptr<int> clone(int p) {
    // ok: 显示地用 int*创建 shared_ptr<int>
    return shared_ptr<int>(new int(p));
}
```
默认情况下，<mark style="color:red;">**一个用来初始化智能指针的普通指针必须指向动态内存，因为智能指针默认使用`delete`释放它所关联的对象**</mark>。
![1a71669cf802d9380c58ac6f78792899](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/25F111B9-9139-40B6-890F-B52B0D00C7AB.png)
![2a0329ecc81f85bba7f036f5210bc31d](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/BD27DF1B-86D0-49C5-90A1-77924A689E3E.png)

**不要混合使用普通指针和智能指针....**

```cpp
// 在函数被调用时ptr被创建并初始化
void process(shared_ptr<int> ptr)
{
    // 使用ptr
} // ptr 离开作用域，被销毁
```
![b444cdf216c696be1af2a303a6a22c69](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/2EC7058E-E500-4A6F-8463-73163EBAA200.png)

**智能指针和异常**
如何确保发生异常后，资源能被正确地释放？
一个简单的确保资源释放的方法就是使用智能指针。

```cpp
void f()
{
    shared_ptr<int> sp(new int(42)); // 分配一个新对象
    // 这段代码抛出一个异常，且在f中未被捕获
} // 在函数结束时，shared_ptr自动释放内存
```
`C`和`C++`两种语言设计的类，通常都要求用户显式地释放所使用的任何资源。而这些类，可能析构函数没有很好的定义。
```cpp
struct destination; // 表示我们正在连接什么
struct connection; // 使用连接所需的信息
connection connect(destination*); // 打开连接
void disconnect(connection); // 关闭给定的连接
void f(destination &d /* other parameters */)
{
    // 获得一个连接，记住使用完后要关闭它
    connection c = connect(&d);
    // 使用连接
    // 如果我们在f退出前忘记调用disconnect，就无法关闭c了
}
```
如果`connection`有一个析构函数，就可以在f结束时由析构函数自动关闭连接。但是`connection`没有析构函数。所以我们使用`shared_ptr`来保证`connection`正确关闭。

**使用我们自己的释放操作**
为了用`shared_ptr`来管理一个`connection`，我们必须首先定义一个函数来代替`delete`。
<mark style="color:red;">这个**删除器(deleter)**函数必须能够完成对`shared_ptr`中保存的指针进行释放操作</mark>。

```cpp
void end_connection(connection *p) { disconnect(*p); }

void f(destination &d /* other parameters */)
{
    connection c = connect(&d);
    shared_ptr<connection> p(&c, end_connection);
    // 使用连接
    // 当f退出时(即使是由于异常而退出)，connection也会正确关闭
}
```

#### 关于`shared_ptr`几点注意事项

1. 不要用裸指针初始化多个`shared_ptr`,会出现`double_free`导致程序崩溃;如:

   ```cpp
   int *iptr = new int(10);
   std::shared_ptr<int> sptr01(iptr);
   std::shared_ptr<int> sptr02(iptr); //错误
   ```

2. <mark style="color:red;">通过`shared_from_this`返回this指针</mark>，不能把this指针作为`shared_ptr`直接返回，因为this指针本质就是裸指针，通过this返回可能导致重复析构，不能把`this`指针交给智能指针管理:

   ```cpp
   class A {
       shared_ptr<A> GetSelf() {
          return shared_from_this();
          // return shared_ptr<A>(this); 错误，会导致double free
       }  
   };
   ```

3. 尽量使用`make_shared`，少用`new`;

4. 不要`delete get()`返回的裸指针;

5. 不是new处理的空间要自定义删除器；

6. 要避免循环引用，循环应用导致内存永远不会被释放，造成内存泄露。

### `unique_ptr`

- `unique_ptr`是独占似的，某个时刻只能有一个`unique_ptr`指向一个给定对象，当`unique_ptr`被销毁时，它所指向的对象也会被销毁。
- `unique_ptr`没有类似`make_shared`的标准库函数。
- **当我们定义一个`unique_ptr`时，需要将其绑定到一个`new`返回的指针上**。
```cpp
unique_ptr<double> p1; // 可以指向一个double的unique_ptr
unique_ptr<int> p2(new int(42)); // p2指向一个值为42的int
```
`unique_ptr` 不支持普通的拷贝或赋值操作。
```cpp
unique_ptr<string> p1(new string("Stegosaurus"));
unique_ptr<string> p2(p1); // error: unique_ptr 不支持拷贝
unique_ptr<string> p3;
p3 = p2; // error: unique_ptr 不支持赋值
```
![f3b6628def95708fff8a5316c27bcd09](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/64C7A7E2-114C-4E7D-8D45-969B8ED47227.png)
通过`release`或`reset`将指针的所有权从一个(非const)`unique_ptr`转移给另一个unique:

```cpp
// 将所有权从p1(指向string Stegosaurus)转移给p2
unique_ptr<string> p2(p1.release()); // release 将 p1 置为空
unique_ptr<string> p3(new string("Trex"));
// 将所有权从p3转移给p2
p2.reset(p3.release()); // reset 释放了p2原来指向的内存
```
`release`会切断`unique_ptr`和它原来管理的对象间的联系。**`release`返回的指针通常用来初始化另一个智能指针或给另一个智能指针赋值**。
但是，**如果我们不用另一个智能指针来保存release返回的指针，我们的程序就要负责资源的释放**:

```cpp
p2.release(); // 错误: p2 不会释放内存，而且我们丢失了指针
auto p = p2.release(); //正确，但我们必须记得delete(p)，说明p此时不是指针
```

**传递`unique_ptr`参数和返回`unique_ptr`**
<mark style="color:red;">不能拷贝`unique_ptr`的规则有一个例外：我们可以拷贝或复制一个将要被销毁的`unique_ptr`。常见的就是从函数返回一个`unique_ptr`</mark>:

```cpp
unique_ptr<int> clone(int p) {
    // ok: 从int*创建一个unique_ptr<int>
    return unique_ptr<int>(new int(p));
}
```
还可以返回一个局部对象的拷贝:
```cpp
unique_ptr<int> clone(int p) {
    unique_ptr<int> ret(new int (p));
    // . . .
    return ret;
}
```

**向`unique_ptr`传递删除器**
类似`shared_ptr`,`unique_ptr`默认情况下用`delete`释放它指向的对象。
来看一个具体的例子:
```cpp
void f(destination &d /* 其他需要的参数 */)
{
    connection c =  connect(&d); // 打开连接
    // 当p被销毁时，连接将会关闭
    unique_ptr<connection,decltype(end_connection)*> p(&c,end_connection);
    // 使用连接
    // 当f退出时(即使由于异常而退出)，connection会被正确关闭
}
```
<mark style="color:red">`decltype(end_connection)`返回的是一个函数类型，所以我们必须添加`*`来指出我们正在使用该类型的一个指针</mark>。

下面的`unique_ptr`声明中，哪些是合法的，哪些可能导致后续的程序报错？
![4cff0c4a35f1c15597a91f671c9e0071](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/F6A8FC60-D390-450B-92B3-54F90521BA82.png)

