## C++ Primer 类设计: 拷贝控制

**拷贝控制操作**

- **拷贝构造函数**
- **拷贝赋值运算符**
- **移动构造函数**
- **移动赋值运算符**
- **析构函数**

`拷贝构造函数`、`移动构造函数`定义了当用同类型的另一个对象初始化本对象时会做什么？
`拷贝赋值运算符`、`移动赋值运算符`定义了讲一个对象赋予同类型的另一个对象时做什么？
`析构函数`定义了此雷翔对象销毁时做什么。

#### 拷贝构造函数
如果一个构造函数的 **第一个参数** 是自身类类型的引用，且**任何额外参数都有默认值**，则此构造函数是拷贝构造函数:
```cpp
class Foo {
    public:
    Foo(); // 默认构造函数
    Foo(const Foo&); // 拷贝构造函数
    // ...
};
```
- 拷贝构造函数的第一个参数必须是 引用类型；
- 拷贝构造函数在几种情况下都会被隐式地使用，因此，拷贝构造函数通常不应该是`explicit`的;

**合成拷贝构造函数**

- 如果我们没有为一个类定义拷贝构造函数，编译器会为我们定义一个。
- 与合成默认构造函数不同，即使我们定义了其他构造函数，编译器也会为我们合成一个 拷贝构造函数;
- 一般时候，通过 **合成拷贝构造函数** 来**阻止**我们拷贝该类型的对象;
- 默认情况下，**合成拷贝构造函数** 会将参数的成员**逐个拷贝**到正在创建的对象中。编译器从给定的对象中依次将每个非`static`成员拷贝到正在创建的对象中；
- 每个成员的类型都决定了它如何拷贝: 对类类型的成员，会使用其拷贝构造函数来拷贝。内置类型的成员直接拷贝；

拷贝初始化不仅在我们用`=`定义变量时发生，在下列情况下也会发生:
- 使用`=`定义一个变量时;
- **将一个对象作为实参传递给一个 非引用类型的形参**;
- **从一个 返回类型为非引用类型 的函数返回对象**；
- 用花括号列表初始化一个数组中的元素 或 一个聚合类中的成员时;

如当我们初始化标准库容器 或 调用其`insert`或`push`成员时，容器会对其元素进行拷贝初始化。与之相对的，用`emplace`成员创建的元素都进行直接初始化。

拷贝构造函数为什么必须是引用类型？
拷贝构造函数被用来 初始化非引用类 类型参数，这一特性解释了为什么 拷贝构造函数自己的参数必须是引用类型。
如果参数不是引用类型，则调用永远不成功 —— 为了调用拷贝构造函数，我们必须拷贝它的实参，但是为了拷贝实参，我们有需要调用拷贝构造函数，如此无限循环。

问题：解释问什么下面的声明是非法的？
`Sales_data::Sales_data(Sales_data rhs)`
答: 参数必须是一个引用。

问题： 假定Point是一个类类型，它有一个public的拷贝构造函数，指出下面程序哪些片段使用了拷贝构造函数？
![6578f1728f7e92d9e8349b7f62ea16df](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/51EA8AB1-A378-4B20-820B-C477208336A0.png)

下面来看一个例子:
```c++
class HasPtr {
public:
    HasPtr(const std::string &s = std::string()):ps(new std::string(s)),i(0) 
    {
        std::cout << "HasPtr common constructor called" << std::endl;
    }
    HasPtr(const HasPtr &ori): ps(new std::string(*ori.ps)),i(ori.i)
    {
        std::cout << "HasPtr copy constructor called" << std::endl;
    }
    
    std::string GetPs();
    int GetI();

    ~HasPtr(){
        std::cout << "HasPtr delete func" << std::endl;
        if(ps != nullptr){
            delete ps;
        }
        ps=nullptr;
    }
private:
    std::string *ps;
    int i;
};

std::string HasPtr::GetPs(){
    return *ps;
}

int HasPtr::GetI(){
    return i;
}

调用:
void hasPtrTest(){
    //HasPtr h1=string("good");
    HasPtr h1("good");
    HasPtr h2=h1;
    cout << "h2.ps=>" << h2.GetPs() << " h2.i=>"<< h2.GetI() << endl;
}
结果:
HasPtr common constructor called
HasPtr copy constructor called
h2.ps=>good h2.i=>0
HasPtr delete func
HasPtr delete func
```

#### 拷贝赋值运算符
与类控制其对象如何初始化一样，类也可以控制其对象如何赋值:
```cpp
Sales_data trans,accum;
trans = accum; // 使用Sales_data的拷贝复制运算符
```
如果 类未定义 自己的拷贝赋值运算符，则编译器会为它合成一个。

**重载赋值运算符**

- 重载运算符本质是一个函数，其名字由`operator`关键字后接表示要定义的 运算符的符号 组成;
- 因此，赋值运算符就是一个名为`operator =`的函数。和其他函数类似，运算符函数也有 返回类型 和 参数列表;
- 某些运算符，包括赋值运算符，必须定义为成员函数。**如果一个运算符是成员函数，则 左侧运算对象 就绑定到隐式的`this`参数**。
```cpp
class Foo{
public:
Foo& operator=(const Foo&); // 赋值运算符
//...
}
```
为了与内置类型的赋值保持一致，**赋值运算符通常返回一个指向其 左侧运算对象的引用**。

**合成拷贝赋值运算符**
和拷贝构造函数一样，如果一个类未定义自己的 拷贝赋值运算符，编译器会为它生产一个 **合成拷贝赋值运算符**。

- **合成拷贝赋值运算符** 可以用来 禁止该类型对象的赋值；
- **合成拷贝赋值运算符** 默认会将右侧运算对象的 每个非static 成员赋予左侧运算对象的对应成员。
```cpp
// 等价于合成拷贝赋值运算符
Sales_data& Sales_data::operator=(const Sales_data &rhs)
{
    bookNo = rhs.bookNo; // calls the string::operator=
    units_sold = rhs.units_sold; // uses the built-in int assignment
    revenue = rhs.revenue; // uses the built-in double assignment
    return *this; // return a reference to this object
}
```
上面`class HasPtr`的示例:
```cpp
HasPtr& operator=(const HasPtr &rhs){
    std::cout<< "operator= called" <<std::endl;
    ps = new std::string(*rhs.ps);
   i=rhs.i;
}
```

#### 析构函数
- 在构造函数中，成员的初始化是在函数体执行之前完成的，且按照它们在类中出现的顺序进行初始化。
- **在析构函数中，首先执行函数体，然后销毁成员，成员按初始化顺序逆序销毁**；
- 销毁类类型的成员需要指向成员自己的析构函数；
- **隐式销毁 一个内置指针类型 的成员不会`delete` 它所指向的对象**;
- **与普通指针不同，智能指针是类类型，所以具有析构函数。因此，与普通指针不同，智能指针成员在析构阶段自动销毁**。
- **认识到 析构函数本身并不直接销毁成员 是非常重要的。成员是在析构函数体之后 隐含的 析构阶段被销毁的**。

### 三/五法则

- **需要析构函数的类，通常也需要 拷贝构造函数 和 拷贝赋值函数**
  a. 当我们决定 一个类是否需要定义自己版本的拷贝控制成员时，一个基本原则是首先确定这个类是否需要 析构函数;
  如果这个类需要析构函数，我们几乎可以肯定它也需要一个拷贝构造函数和拷贝复制运算符。
  (但并不代表，如果这个类需要拷贝构造/赋值函数，就一定需要析构函数。比如类成员中有`shared_ptr<vector<T>>`,此时类并不需要析构函数，单需要拷贝构造/赋值函数)
  如上面的`class HasPtr`，如果使用 合成的拷贝构造/赋值运算符，则意味着多个`HasPtr`对象可能指向相同的内存:

  ```cpp
  HasPtr f(HasPtr hp) // HasPtr 是传值参数，所以将被拷贝
  {
      HasPtr ret=hp; // 拷贝给定的HasPtr
      return ret; // ret 和 hp被销毁
  }
  ```

  当`f`返回时，`hp`和`ret`都被销毁，在两个对象上都会调用`HasPtr`的析构函数。次析构函数会`delete ret`和`hp`中的指针成员，但两个对象包含相同的指针值，因此会导致此指针被`delete`两次。
- **需要拷贝操作的类，也需要赋值操作，反之亦然**
  例如，一个类为每个对象分配一个独有的、唯一的序号。这个类需要一个拷贝构造函数 为 每个新建的对象生成一个新的、独一无二的序号。除此之外，这个拷贝构造函数从给定对象拷贝所有其他数据成员。
  这个类还需要自定义拷贝赋值运算符来避免将序号赋予目的对象。但是，这个类并不需要析构函数。

#### 使用`=default`
a. 通过将拷贝控制成员定义为`=default`来显式要求编译器生成合成的版本。
b. 当我们在类内用`=default`修饰成员的声明时，合成的函数将隐式地声明为内联的。
如果我们不希望合成的成员是内联函数，应该只对成员的类外定义使用`=default`；
![b02e8974d3ae138ec916230d6c920060](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/A3CEB506-30DC-4D6F-93F4-93DF80396D58.png)

### 阻止拷贝
如:`iostream`类阻止了拷贝，以避免多个对象写入 或 读取相同的IO缓冲。

**定义删除的函数`=delete`**

```cpp
Struct NoCopy{
    NoCopy() = default; // 使用合成的默认构造函数
    NoCopy(const NoCopy &) = delete; //阻止拷贝
    NoCopy &operator=(const NoCopy&)=delete; //阻止赋值
    ~NoCopy() = default; //使用合成的析构函数
    //其他成员
};
```

**析构函数不能是删除函数**

#### `private`拷贝控制
在新标准发布之前，类是通过将拷贝构造函数 和 拷贝赋值运算符 **声明为`private`来阻止拷贝**:
```cpp
class PrivateCopy{
    // 拷贝控制成员是private的，因此普通用户代码无法访问
    PrivateCopy(const PrivateCopy &);
    PrivateCopy &operator=(const Private&);
    // 其他成员
public:
    PrivateCopy() = default; //合成的默认构造函数
    ~PrivateCopy(); // 用户可以定义此类对象，但是无法拷贝他们
};
```
为了阻止 友元 和 成员函数进行拷贝，我们将拷贝控制成员声明为`private`的，同时并不定义他们。
(阻止用户拷贝的方式是`private`、阻止成员函数、友元拷贝的方法是 **只声明不定义**)

#### 拷贝控制和资源管理
一般来说，有两种选择：可以定义拷贝操作，使类的行为看起来像一个值 或 像一个指针。
- 类的行为像一个值，意味着它有自己的状态。当我们拷贝一个像值的对象时，副本和源对象是完全对立的；
- 行为像指针的类则共享状态，当我们拷贝一个这种累的对象时，副本和原对象使用相同的底层数据。
- 使用标准库容器和`string`类的行为像一个值，而`shared_ptr`类提供类似指针的行为;

##### 行为像值的类
```cpp
class HasPtr {
public:
    HasPtr(const std::string &s = std::string()):
        ps(new std::string(s)), i(0) { }
    // each HasPtr has its own copy of the string to which ps points
    HasPtr(const HasPtr &p):
        ps(new std::string(*p.ps)), i(p.i) { }
    HasPtr& operator=(const HasPtr &);
    ~HasPtr() { delete ps; }
private:
    std::string *ps;
    int i;
};
```

**类值 拷贝赋值运算符(先释放，再构造)**
a. 赋值运算符 通常组合了 析构函数和构造函数的操作;
b. 赋值操作会销毁左侧运算对象的资源；
c. 通过先拷贝右侧运算对象，我们可以处理 **自赋值情况**。并保证在发生异常时代码是安全的
在完成拷贝后，我们释放左侧计算对象的资源，并更新指针指向新分配的`string`:

```cpp
HasPtr& HasPtr::operator=(const HasPtr &rhs)
{
    auto newp = new string(*rhs.ps); // 拷贝底层string
    delete ps; // 释放旧内存
    ps = newp; // 从右侧运算对象拷贝数据到本对象
    i = rhs.i;
    return *this; // 返回本对象
}
```
当我们编写赋值运算符时，有两点需要记住:
a. **如果将一个对象赋予它自身，赋值运算符必须能正确工作**；
b. **大多数赋值运算符组合了 析构函数 和 拷贝构造函数的工作**;

我们来看下如果赋值运算符写成下面这样，自赋值时会发生什么？
```cpp
// 这样编写赋值运算符是错误的
HasPtr& HasPtr::operator=(const HasPtr &rhs)
{
    delete ps; // 释放对象指向的string
    // 如果rhs 和 *this是同一个对象，我们就将从已释放的内存中拷贝数据
    ps = new string(*(rhs.ps));
    i = rhs.i;
    return *this;
}
```

##### 定义行为像指针的类
下面例子中，析构函数不能单方面地释放关联的`string`，只有当最后一个指向`string`的`HasPtr`销毁时，它才可以释放string。
- 令一个类展现类似指针的行为最好的方法是使用`shared_ptr`来管理类中资源;
- 拷贝(或赋值)一个`shared_ptr`会拷贝`shared_ptr`所指向的指针；
- `shared_ptr`类自己记录有多少用户共享它所指向的对象。

**定义一个使用引用计数的类:**

```cpp
class HasPtr {
public:
    // constructor allocates a new string and a new counter, which it sets to 1
    HasPtr(const std::string &s = std::string()):
        ps(new std::string(s)), i(0), use(new std::size_t(1)) {}
    // copy constructor copies all three data members and increments the counter
    HasPtr(const HasPtr &p):
        ps(p.ps), i(p.i), use(p.use) { ++*use; }
    HasPtr& operator=(const HasPtr&);
    ~HasPtr();
private:
    std::string *ps;
    int i;
    std::size_t *use; // member to keep track of how many objects share *ps
};
HasPtr::~HasPtr()
{
    if (--*use == 0) { // if the reference count goes to 0
        delete ps; // delete the string
        delete use; // and the counter
    }
}
HasPtr& HasPtr::operator=(const HasPtr &rhs)
{
    ++*rhs.use; // increment the use count of the right-hand operand
    if (--*use == 0) { // then decrement this object's counter
        delete ps; // if no other users
        delete use; // free this object's allocated members
    }
    ps = rhs.ps; // copy data from rhs into this object
    i = rhs.i;
    use = rhs.use;
    return *this; // return this object
}
```

### 交换操作
a. 除了定义拷贝控制成员，管理资源的类通常还需要定义一个名为`swap`的函数;
b. 对于那些与 重排元素顺序 的算法一起使用的类，`swap`非常重要。这类算法需要交换两个元素时会调用`swap`;
c. 如果一个类定义了自己的`swap`,那么算法将使用类自定义版本。否则，算法将使用标准库定义的`swap`。
如交换两个类值`HasPtr`对象的代码可能像下面这样:

```cpp
HasPtr temp = v1; // 创建v1的值的临时副本
v1 = v2; // 将v2的值赋予v1
v2 = temp; // 将保存的v1的值赋予 v2
```
d. 上面`swap`操作涉及多次内存分配，而这些内存分配都是不必要的，我们更希望`swap`交换指针:
```cpp
string *temp = v1.ps;
v1.ps = v2.ps;
v2.ps = temp;
```

**编写我们自己的`swap`函数**
```cpp
class HasPtr {
    friend void swap(HasPtr&, HasPtr&);
    // 其他成员函数定义
};
inline
void swap(HasPtr &lhs, HasPtr &rhs)
{
    using std::swap;
    swap(lhs.ps, rhs.ps); // 交换指针，而不是string数据
    swap(lhs.i, rhs.i); // 交换int成员
}
```
注意: 与拷贝控制成员不同，swap并不是必要的。**但是对于，分配了资源的类，定义swap可能是一种很重要的优化手段**。

**`swap`函数应该调用`swap`，而不是`std::swap`**
a. 上面例子中，因为数据成员是内置类型的，所以我们调用`std::swap`无可厚非;
b. 但，如果一个类的成员有 自己类型 特定的swap函数，此时调用`std::swap`就是一个错误；
如：一个名为`Foo`的类，有一个类型为`HasPtr`的成员`h`。我们可以为`Foo`编写一个`swap`函数，来避免拷贝:

```cpp
void swap(Foo &lhs,Foo &rhs){
    // 错误：这个函数使用了标准库版本的swap，而不是HasPtr版本
    std::swap(lhs.h,rhs.h);
    // 交换类型Foo的其他成员
}
```
正确的写法:
```cpp
void swap(Foo &lhs,Foo &rhs){
    using std::swap;
    swap(lhs.h,rhs.h); // 使用HasPtr版本的swap
    // 交换类型Foo的其他成员
}
```
上面这个代码做了很好的兼容，如果类有自己的swap，则匹配程度会优先于`std`定义的版本。如果不存在类特定的版本，则会使用`std`版本的`swap`。

**在赋值运算符中使用swap(很重要)**
a. 定义`swap`的类通常用`swap`来定义他们的赋值运算符，这些运算符使用了一种名为 **拷贝并交换的技术**

```cpp
// 注意rhs是按值传递的，意味着HasPtr的拷贝构造函数
// 将右侧运算符中的string 拷贝到 rhs
HasPtr& HasPtr::operator=(HasPtr rhs)
{
    // 交换左侧运算对象和局部变量rhs的内容
    swap(*this,rhs); // rhs现在指向本对象曾经使用的内存
    return *this; // rhs被销毁，从而delete了rhs中的指针
}
```
一定注意: **`rhs`是值传递。该函数自赋值情况下也是正确的。它通过在改变左侧运算对象之前拷贝右侧运算对象保证了自赋值的正确，这与我们再原来的赋值运算符中思路一脉相承**。

### 对象移动
很多情况下，都会发生对象拷贝，但是在某些情况下，对象拷贝后就立即销毁了。这种情况下，移动而非拷贝对象会大幅提升性能。

a. 如前面的`strVec`类中，重新分配内存的过程中，从旧内存将元素拷贝到新内存是不必要的，更好的方式是移动元素;
b. **使用移动而不是拷贝的另一个原因源于IO类 或 `unique_ptr`这样的类，这些类都包含不能被共享的资源(如指针或IO缓冲)。因此,这些类型的对象不能拷贝但可以移动**;
c. 标准库容器、string 和 `shared_ptr`类既支持移动也支持拷贝。**IO类和`unique_ptr`类可以移动但不能拷贝**;

#### 右值引用
a. 新标准引入新的引用类型——**右值引用(rvalue reference)**;
b. 通过`&&`而不是`&`来获得右值引用;
c. 右值引用一个很重要的性质——**只能绑定到一个将要销毁的对象上**;

```cpp
int i = 42;
int &r = i; // ok: r 引用 i
int &&rr = i; // error: 不能将一个右值引用绑定到一个左值上
int &r2 = i * 42; // error: i * 42 是一个右值
const int &r3 = i * 42; // ok: 我们可以讲一个const的引用绑定到一个右值上
int &&rr2 = i * 42; // ok: 将rr2绑定到乘法结果上
```
d. 返回左值引用的函数，连同赋值、下标、解引用和前置递增/递减运算符，都是返回左值的表达式的例子;
e. 返回非引用类型的函数，连同算术、关系、位以及后置递增/递减运算符，都是生成右值；
f. 可以将一个const的左值引用或一个右值引用绑定到这类表达式上;

**左值持久，右值短暂**
右值引用只能绑定到临时对象，我们得知:
a. 所引用的对象将要销毁;
b. 该对象没有其他用户;
也就是说：使用右值引用的代码可以自由地接管所引用的对象的资源。

**变量是左值**
```cpp
int &&rr1 = 42; //正确: 字面常量是右值
int &&rr2 = rr1; //错误: 表达式rr1是左值
```

**标准库move函数**
虽然不能将一个右值引用直接绑定到一个左值上，但我们可以显式地将一个左值转换为对应的右值引用类型。
`int &&rr3=std::move(rr1);// ok`
调用`move`就意味着：除了对rr1赋值 或 销毁外，我们将不再适用它。调用`move`后，我们不能对移后源对象的值做任何假设。

#### 移动构造函数和移动赋值运算符
- 这两个操作类似对应拷贝，但他们从给定对象"窃取"而不是拷贝资源;
- 移动构造函数 第一个参数是一个右值引用，任何额外的参数都必须有默认实参。
- **移动构造函数 必须确保移后源对象处于这样一个状态—— 销毁它是无害的。特别是，一旦资源完成移动，源对象必须不再指向被移动的资源，这些资源的所有权已经归属新创建的对象**。
```cpp
StrVec::StrVec(StrVec &&s) noexecpt // 移动操作不应抛出任何异常
// 成员初始化器接管s中资源
: elements(s.elements),first_free(s.first_free),cap(s.cap)
{
   // 令s进入这样的状态——对其运行析构函数是安全的
   s.elements = s.first_free = s.cap = nullptr;
}
```
a. `noexcept`通知标准库我们的构造函数不抛出任何异常;
```cpp
class StrVec {
public:
    StrVec(StrVec&&) noexcept; // move constructor
    // other members as before
};
StrVec::StrVec(StrVec &&s) noexcept : /* member initializers */
{ /* constructor body */ }
```

**移动赋值运算符**

```cpp
StrVec &StrVec::operator=(StrVec &&rhs) noexcept
{
    // 直接检测自赋值
    if (this != &rhs) {
        free(); //释放已有元素
        elements = rhs.elements; //从rhs接管资源
        first_free = rhs.first_free;
        cap = rhs.cap;
        // 将rhs置于可析构状态
        rhs.elements = rhs.first_free = rhs.cap = nullptr;
    }
    return *this;
}
```
检查自赋值情况原因是：此右值可能是`move`调用的返回结果。

**移动右值，拷贝左值**

```cpp
StrVec v1,v2;
v1 = v2; // v2是左值；使用拷贝赋值
StrVec getVec(istream &); // getVec返回一个右值
v2 = getVec(cin); // getVec(cin)是一个右值，使用移动赋值
```

**如果没有移动构造函数，右值也可以被拷贝**
```cpp
class Foo {
public:
    Foo() = default;
    Foo(const Foo&); // 拷贝构造函数
    // 其他成员定义，但Foo未定义移动构造函数
};
Foo x;
Foo y(x); // 拷贝构造函数；x是一个左值
Foo z(std::move(x)); // 拷贝构造函数，因为未定义移动构造函数
```
在对`z`进行初始化时，我们调用`move(x)`，它返回一个绑定x的`Foo&&`。`Foo`的拷贝构造函数是可行的，因为我们将一个`Foo&&`转换为一个`const Foo&`。

### explicit

文档原文: [https://github.com/Light-City/CPlusPlusThings/tree/master/basic_content/explicit](https://github.com/Light-City/CPlusPlusThings/tree/master/basic_content/explicit)

- `explicit` 修饰构造函数时，可以防止隐式转换和复制初始化;
- `explicit` 修饰转换函数时，可以防止隐式转换，但按语境转换除外;

```cpp
struct A
{
    A(int) { }
    operator bool() const { return true; }
};
struct B
{
    explicit B(int) {}
    explicit operator bool() const { return true; }
};
void doA(A a) {}
void doB(B b) {}
int main()
{
    A a1(1);        // OK：直接初始化
    A a2 = 1;        // OK：复制初始化
    A a3{ 1 };        // OK：直接列表初始化
    A a4 = { 1 };        // OK：复制列表初始化
    A a5 = (A)1;        // OK：允许 static_cast 的显式转换 
    doA(1);            // OK：允许从 int 到 A 的隐式转换
    if (a1);        // OK：使用转换函数 A::operator bool() 的从 A 到 bool 的隐式转换
    bool a6(a1);        // OK：使用转换函数 A::operator bool() 的从 A 到 bool 的隐式转换
    bool a7 = a1;        // OK：使用转换函数 A::operator bool() 的从 A 到 bool 的隐式转换
    bool a8 = static_cast<bool>(a1);  // OK ：static_cast 进行直接初始化

    B b1(1);        // OK：直接初始化
//    B b2 = 1;        // 错误：被 explicit 修饰构造函数的对象不可以复制初始化
    B b3{ 1 };        // OK：直接列表初始化
//    B b4 = { 1 };        // 错误：被 explicit 修饰构造函数的对象不可以复制列表初始化
    B b5 = (B)1;        // OK：允许 static_cast 的显式转换
//    doB(1);            // 错误：被 explicit 修饰构造函数的对象不可以从 int 到 B 的隐式转换
    if (b1);        // OK：被 explicit 修饰转换函数 B::operator bool() 的对象可以从 B 到 bool 的按语境转换
    bool b6(b1);        // OK：被 explicit 修饰转换函数 B::operator bool() 的对象可以从 B 到 bool 的按语境转换
//    bool b7 = b1;        // 错误：被 explicit 修饰转换函数 B::operator bool() 的对象不可以隐式转换
    bool b8 = static_cast<bool>(b1);  // OK：static_cast 进行直接初始化

    return 0;
}
```

