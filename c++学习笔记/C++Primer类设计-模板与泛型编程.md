## C++Primer 类设计: 模板与泛型编程

- 模板是C++泛型编程的基础;
- 一个模板就是一个创建类 或 函数的蓝图 或者说公式;
- `vector`就是一个泛型类型，`find`就是泛型函数；

### 函数模板
- 函数模板就是一个公式，可以用来生成针对特定类型的函数版本;
- 模板定义关键字`template`开始，后跟一个**模板参数列表(template parameter list)**;
- 模板参数列表的作用很像 函数参数列表； 函数参数列表定义了若干特定类型的的局部变量，但并未指出如何初始化它们。在运行时，调用者提供实参初始化形参;
- 模板参数表示在类 或 函数定义中用到的 类型 或 值。当使用模板时，我们指定 模板实参，将其绑定到 模板参数上;
- 下面例子中，`T`表示的实际类型在编译时根据`compare`的使用情况来确定；
```cpp
template <typename T>
int compare(const T &v1,const T &v2){
    if (v1 < v2) return -1;
    if (v2 < v1) return 1;
    return 0;
}
```

**实例化函数模板**

- 当我们调用一个函数模板时，编译器(通常)用函数实参来为我们推断模板实参；
- 也就是当我们调用`compare`时，编译器使用实参的类型来确定 绑定到模板参数`T`的类型;
```cpp
// 实例化出 int compare(const int &, const int&)
cout << compare(1,0) << endl; // T为int
// 实例化出 int compare(const vector<int>&, const vector<int>&)
vector<int> vec1{1,2,3}, vec2{4,5,6};
cout<< compare(vec1,vec2); // T为vector<int>
```

**模板类型参数**
`compare`函数有一个模板类型参数(type parameter)。
我们将类型参数看做类型说明符，就像内置类型 或 类类型说明符一样使用。
类型参数可以用来指定 返回类型 或 函数的参数类型，以及在函数体内用于变量声明 或 类型转换。
类型参数前必须使用 关键字 `class` 或 `typename`。

```cpp
template <typename T>
T foo(T *p)
{
    T tmp = *p; // tmp的类型将是指针p指向的类型
    // ....
    return tmp;
}
```
```cpp
// 正确: 在模板参数列表中，typename 和 class 没有什么区别
template <typename T, class U> calc (const T&, const U&);
```

**非类型模板参数(不常用吧)**
一个 非类型参数 表示一个值而非一个类型。我们通过特定的类型名而非关键字 class或 typename 来指定非类型的参数。
下面这个例子，比较不同长度的字符串字面常量，因此为模板定义了两个非类型的参数。
第一个模板参数表示 第一个数组的长度；
第二个模板参数表示 第二个数组的长度。

```cpp
template<unsigned N,unsigned M>
int compare(const char (&p1)[N], const char (&p2)[M]){
    return strcmp(p1,p2);
}
```
调用:
```cpp
compare("hi","mom");
```
编译器会使用字面常量的大小来代替N 和 M，同时编译器会在一个字符串字面常量的末尾插入一个空字符作为终结符。
所以编译器实例化出如下版本:
```c++
int compare(const char (&p1)[3], const char (&p2)[4]);
```
注意: **非类型模板参数的模板实参必须是 常量表达式**

**`inline`和`constexpr`的函数模板**
```cpp
// 正确: inline 说明符跟在模板参数列表之后
template <typename T> inline T min(const T&, const T&);
// 错误: inline说明符位置不正确
inline template <typename T> T min(const T&, const T&);
```

**模板编译**
为了生成一个实例化版本，编译器需要掌握函数模板 或 类模板成员函数的定义。
因此，模板的头文件通常包括 声明 也包括 定义。

### 类模板
**与函数模板不同的是，编译器不能为 类模板 推断模板参数类型。所以我们必须在模板名后的尖括号中提供额外的信息** —— 用来代替模板参数的模板实参列表。

```cpp
#ifndef BLOB_H
#define BLOB_H
template <typename T>
class Blob {
public:
    typedef T value_type;
    typedef typename std::vector<T>::size_type size_type;
    // 构造函数
    Blob();
    Blob(std::initializer_list<T> il);
    // Blob中的元素数目
    size_type size() const { return  data->size(); }
    bool empty() const { return data->empty(); }
    // 添加和删除元素
    void push_back(const T &t){
        data->push_back(t);
    }
    void push_back(T &&t){
        data->push_back(std::move(t));
    }
    void pop_back();
    // 元素访问
    T& back();
    T& operator[](size_type i);

    void print(){
        for(auto item:*data){
            std::cout<<"item =>" << item << std::endl;
        }
    }
private:
    std::shared_ptr<std::vector<T>> data;
    // 若data[i]无效，则抛出msg
    void check(size_type i,const std::string &msg) const;
};

template <typename T>
void Blob<T>::pop_back(){
    check(0,"pop_back on empty Blob");
    data->pop_back();
}
```
**实例化类模板**
```c++
Blob<int> ia; // 空Blob<int>
Blob<int> ia2={0,1,2,34}; // 有5个元素的Blob<int>
```
`ia`和`ia2`使用相同的特定类型版本的`Blob`即(`Blob<int>`)。

**对我们指定的每种元素类型，编译器都生成一个不同的类:**
```c++
// 下面的定义实例化两个不同的Blob类型
Blob<string> names; // 保存string的Blob
Blob<double> prices; // 不同的元素类型
```

**类模板的成员函数**
a. 类模板的成员函数具有和模板相同的模板参数;
b. 定义在类模板之外的成员函数就必须以关键字template开始，后接类模板参数列表;

```cpp
template <typename T>
void Blob<T>::check(size_type i, const std::string &msg) const
{
    if(i >= data->size())
        throw std::out_of_range(msg);
}
template <typename T>
T& Blob<T>::back()
{
    check(0,"back on empty Blob");
    return data->back();
}
template <typename T>
T& Blob<T>::operator[](size_type i)
{
    check(i,"back on empty Blob");
    return (*data)[i];
}
```
**Blob构造函数**
```cpp
//构造函数
template <typename T>
Blob<T>::Blob(): data(std::make_shared<std::vector<T>>())
{}

template <typename T>
Blob<T>::Blob(std::initializer_list<T> il):data(std::make_shared<std::vector<T>>(il))
{}
```
使用上面的构造函数，我们可以类似下面实例化Blob类型
`Blob<string> articles= {"a","an","the"};`

**类模板成员函数的实例化**
如果一个成员函数没有被使用，则它不会被实例化。成员函**只有在被用到时才进行实例化**
下面这段代码实例化了`Blob<int>`类和它的三个成员函数:`operator[]`、`size`、接受`initializer_list<int>`的构造函数。

```cpp
Blob<int> squares = {0,1,2,3,4,5};
for(size_t i=0; i!=squares.size(); ++i)
    squares[i] = i*i; // 实例化 Blob<int>::operator[](size_t)
```

**类代码内简化模板类名的使用**
在类模板自己的作用域中，我们可以直接使用模板名而不提供实参:

```cpp
template <typename T> 
class BlobPtr {
public:
    BlobStr():curr(0) {}
    BlobPtr(Blob<T> &a, size_t sz =0): wptr(a.data),curr(sz) {}
    T& operator* () const
    {
        auto p = check(curr,"dereference past end");
        return (*p)[curr]; // (*p)为本对象指向的vector
    }
    // 递增和递减
    BlobPtr& operator++(); //前置运算符
    BlobPtr& operator--();
private:
    //若检查成功,check返回一个指向vector的shared_ptr
    std::shared_ptr<std::vector<T>> check(std::size_t, const std::string &) const;
    //保存一个weak_ptr,表示底层vector可能被销毁
    std::weak_ptr<std::vector<T>> wptr;
    std::size_t curr; //数组中当前位置
};
#endif
```
我们仔细看`BlobPtr& operator++();`、`BlobPtr& operator--();` 返回值是`BlobPtr&`，而不是`BlobPtr<T>&`。

**在类模板外使用类模板名**
当我们在类模板外定义其成员时，必须记住，我们并不在类作用域中，直到遇到类名才进入类的作用域。
```cpp
template <typename T>
BlobPtr<T> BlobPtr<T>::operator++(int){
    // 此处无需检查; 调用前置递增时会进行检查
    BlobPtr ret = * this; // 保存当前值 
    ++*this; //  推进一个元素;前置++检查递增是否合法
    return ret; // 返回保存状态
}
```
此处返回类型位于类的作用域之外，所以用`BlobPtr<T>`。但是`ret`在类的作用域中，所以可以直接用:`BlobPtr ret = * this;`。

#### 类模板和友元(有点难)
如果类模板包含一个非模板的友元，则友元被授权可以访问所有类模板实例，如`Blob<int>`、`Blob<string>`都可以;
如果友元自身是模板，类可以授权给所有友元模板实例，也可以只授权给特定的实例。

**一对一友好关系**
如我们的Blob类应该将 `BlobPtr`类 和 一个模板版本的`Blob`相等运算符 定义为友元。
为了引用模板的特定实例，我们必须首先声明模板自身。一个模板声明包括模板参数列表:
```cpp
// 前置声明，在Blob中声明友元所需要的
template <typename> class BlobPtr;
template <typename> class Blob; //运算符==中的参数所需要的
template <typename T>
    bool operator==(const Blob<T>&,const Blob<T>&);

template <typename T>
class Blob {
    //每个Blob实例将访问权限授予用相同类型实例化的BloPtr和相等运算符
    friend class BlobPtr<T>;
    friend bool operator==<T>(const Blob<T>&, const Blob<T>&);
    //其他成员定义相同
};
```
a. 我们首先将`Blob`、`BlobPtr`、`operator==`声明为模板，这些声明是`operator==`函数的参数声明以及`Blob`中的友元声明所需要的;
b. 友元的声明用Blob的模板形参作为它们自己的模板实参。因此，友好关系被限定在**用相同类型实例化的Blob和BlobPtr相等运算符之间**
```cpp
Blob<char> ca; // BlobPtr<char> 和 operator==<char> 都是本对象的友元
Blob<int> ia; // BlobPtr<int> 和 operator==<int> 都是本对象的友元
```
a. `BlobPtr<char>`的成员可以访问`ca`(或任何其他`Blob<char>`对象)的非public部分;
b. 但`ca`(`Blob<char>`) 对`ia`(`Blob<int>`) 或 Blob的任何其他实例都没有特殊访问权限;

**通用和特定的模板友好关系**
一个类也可以将另一个模板的每个实例都声明为自己的友元，或限制特定的实例为友元。

### 补充

#### 模板参数可变: 

- **参数个数(variable number): 利用参数个数逐一递减的特性, 实现递归函数调用。使用function template完成**;

- **参数类型(different type): 利用参数个数逐一递减 导致 参数类型也逐一递减的特性，实现递归继承和递归复合。使用class template完成**

- 一般形式:

  ```cpp
  void func() { /*...*/}
  
  template<typename T,typename... Types>
  void func(const T& firstArgs,const Types&... args) //点点点在前头
  {
  	处理 firstArg
  	func(args...); //点点点在后头
  }
  ```

  <mark style="color:red">**使用`sizeof...(args)`获得参数的个数**</mark>

- 示例一

  ```cpp
  **如果有一个或多个参数传入，则使用函数模板，并持续递归。如果没有参数传入，则调用`printX()`普通函数。**
  
  void printX() {}
  
  template <typename T,typename... Types>
  void printX(const T& firstArg, const Types&... args){
    cout<< firstArgs<<endl;
    printX(args...);
  }
  
  示例: printX(7.5,"hello",bitset<16>(377),42);
  输出:
  7.5
  hello
  00000000000011011110001
  42
  ```

  **如果有一个或多个参数传入，则使用函数模板，并持续递归。如果没有参数传入，则调用`printX()`普通函数。**

- 示例二

  参考问题: [https://stackoverflow.com/questions/3634379/variadic-templates](https://stackoverflow.com/questions/3634379/variadic-templates)

  ```cpp
  void printf(const char *s)
  {
    while (*s)
    {
      if (*s == '%' && *(++s) != '%')
        throw std::runtime_error("invalid format string: missing arguments");
      std::cout << *s++;
    }
  }
  
  template<typename T, typename... Args>
  void printf(const char* s, T value, Args... args)
  {
    while (*s)
    {
      if (*s == '%' && *(++s) != '%') //不会管%d、%s这些控制符号,反正值都是朝 std::cout里面丢
      {
        std::cout << value;
        printf(++s, args...); // call even when *s == 0 to detect extra arguments
        return;
      }
      std::cout << *s++;
    }
    throw std::logic_error("extra arguments provided to printf");
  }
  ```

- 示例三:

  ```cpp
  int maximum(int n)
  {
      return n;
  }
  
  template<typename... Args>
  int maximum(int n, Args... args)
  {
      return std::max(n, maximum(args...));
  }
  调用方式: std::cout<<max(57,48,60,100,20)<<std::endl;
  
  如果类型相同，其实用 std::initializer_list<>更合适
  如: std::cout<<std::max({57,48,60,100,20})<<std::endl; 就是用的是 std::initializer_list
  ```

  

