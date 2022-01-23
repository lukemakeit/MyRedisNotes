## C++Primer基础:函数、重载、函数指针

### 函数

- 使用引用避免拷贝:

  ```cpp
  bool isShorter(const string &s1,const string &s2){
      return s1.size() < s2.size();
  }
  ```

- 不能把`const`对象、字面值或者需要类型传回的对象传递个普通的引用形参

  如下面的函数:

  `string::size_type find_char(string &s,char c,string::size_type &occurs);`

  下面这种调用就是产生错误:

  `find_char("Hello world",'o',ctr);`

  再比如下面这种更难察觉的问题，假如其他函数(正确地)将它们的形参定义成常量引用,那么此时将无法使用`find_char`函数:

  ```cpp
  bool is_sentence(const string &s)
  {
      // if there's a single period at the end of s, then s is a sentence
      string::size_type ctr = 0;
      return find_char(s, '.', ctr) == s.size() - 1 && ctr == 1;
  }
  ```

- `main`处理命令行选项

  ```cpp
  int main(int argc,char *argv[]) { ... }
  ```

  `argc`: 表示数组中字符串的数量;

  `argv`: 是一个数组，它的元素是指向C风格字符串的指针;

  如果我们运行的程序命令是:`prog -d -o ofile data0`,则:

  ```cpp
  argv[0] = "prog"; // or argv[0] might point to an empty string
  argv[1] = "-d";
  argv[2] = "-o";
  argv[3] = "ofile";
  argv[4] = "data0";
  argv[5] = 0;
  ```

  注意使用argv中的实参时，一定要记得可选的实参从`argv[1]`开始，`argv[0]`保存程序的名字，而非用户输入。

- **含有可变形参的函数**

  C+11 新标准提供了两种主要方法：

  - 如果所有实参类型相同，可以传递一个名为`initializer_list`的标准库类型;

  - 如果实参类型不同，我们可以编写一种特殊的函数，也就是所谓的可变参数模板(后续介绍);

    ![image-20211128155036491](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211128155036491.png)

    和`vector`不一样的是，`initializer_list`对象中元素永远是常量值，我们无法改变`initializer_list`对象中元素的值。

    示例一:

    ```cpp
    void error_msg(initializer_list<string> il){
        for (auto beg=il.begin();beg != il.end();++beg)
        {
            cout << *beg << " ";
        }
        cout << endl;
        
    }
    传参时，我们还可以如下传入:
    // expected, actual are strings
    if (expected != actual)
        error_msg({"functionX", expected, actual});
    else
        error_msg({"functionX", "okay"});
    ```

    示例二:

    含义`initializer_list`形参的函数也可以同时拥有其他形参。

    如: `ErrCode`用于表示不同类型的错误

    ```cpp
    void error_msg(ErrCode e, initializer_list<string> il)
    {
        cout << e.msg() << ": ";
        for (const auto &elem : il)
        cout << elem << " " ;
        cout << endl;
    }
    使用示例:
    if (expected != actual)
        error_msg(ErrCode(42), {"functionX", expected, actual});
    else
        error_msg(ErrCode(0), {"functionX", "okay"});
    ```

- 有返回值的函数

  - 不要返回局部对象的引用 或 指针

    函数完成后，它所占用的存储空间也随之被释放掉了。因此函数终止意味着局部变量的引用将指向不再有效的内存区域。

    下面这个函数 返回 局部变量的引用 是不可用的。

    ```cpp
    // disaster: this function returns a reference to a local object
    const string &manip()
    {
        string ret;
        // transform ret in some way
        if (!ret.empty())
            return ret; // WRONG: returning a reference to a local object!
        else
            return "Empty"; // WRONG: "Empty" is a local temporary string
        }
    }
    ```

  - 引用返回左值

    调用一个返回引用的函数得到左值，其他返回类型得到右值。如下:

    ```cpp
    char &get_val(string &str, string::size_type ix)
    {
        return str[ix]; // get_val assumes the given index is valid
    }
    int main()
    {
        string s("a value");
        cout << s << endl; // prints a value
        get_val(s, 0) = 'A'; // changes s[0] to A
        cout << s << endl; // prints A value
        return 0;
    }
    ```

- 函数重载

  同一个作用域中几个函数**名字相同**但**形参列表不同**，我们称之为**重载(overloaded)函数**。

  ```cpp
  void print(const char *cp);
  void print(const int *beg, const int *end);
  void print(const int ia[], size_t size);
  
  使用实例:
  int j[2] = {0,1};
  print("Hello World"); // calls print(const char*)
  print(j, end(j) - begin(j)); // calls print(const int*, size_t)
  print(begin(j), end(j)); // calls print(const int*, const int*)
  ```

  - 对应重载函数来说，它们应该在 **形参数量** 或 **形参类型**上有所不同;

  - 不允许两个函数 除了返回类型外其他所有要素都相同。如下面就是错误示例:

    ```cpp
    Record lookup(const Account&);
    bool lookup(const Account&); // error: only the return type is different
    ```

- 默认实参

  如我们使用string对象表示窗口的内容，一般情况下，我们希望该窗口的高、宽和背景字符都是默认值。

  但是我们允许用户为这几个参数自定义指定与默认值不同的数值。

  ```cpp
  typedef string::size_type sz; // typedef see § 2.5.1 (p. 67)
  string screen(sz ht = 24, sz wid = 80, char backgrnd = ' ');
  ```

  注意: 一旦某个形参被赋予了默认值，它后面的所有形参必须有默认值。

  使用默认实参调用函数

  ```cpp
  string window;
  window = screen(); // equivalent to screen(24,80,' ')
  window = screen(66);// equivalent to screen(66,80,' ')
  window = screen(66, 256); // screen(66,256,' ')
  window = screen(66, 256, '#'); // screen(66,256,'#')
  ```

- 内联函数和constexpr函数

  一次函数调用其实包含着一系列工作：调用前保存寄存器，返回时恢复；可能需要拷贝实参；程序转向一个新的位置继续执行。

  - 内联(inline)函数可避免函数调用的开销

    将函数指定为**内联函数(inline)**，通常就是将它在某个调用点上 "内联地展开"。假设我们把`shortString`函数定义为内联函数，则如下调用: `cout << shorterString(s1, s2) << endl;`

    将在编译过程中展开类似于下面的形式:`cout << (s1.size() < s2.size() ? s1 : s2) << endl;`

    从而消除了`shortString`函数运行时的开销。

    在`shortString`函数的返回类型前加上关键字`inline`就可以将它声明为内联函数了:

    ```cpp
    // inline version: find the shorter of two strings
    inline const string &shorterString(const string &s1, const string &s2)
    {
        return s1.size() <= s2.size() ? s1 : s2;
    }
    ```

  - **constexpr函数**

    `constexpr`函数是指能用于常量表达式的函数。

    定义`constexpr`函数方法与其他函数类似，不过需要遵循几项约定: 函数的 返回类型 与 所有形参的类型都得是 字面值类型，而且函数中必须 有且只有一条 return 语句:

    ```cpp
    constexpr int new_sz() { return 42; }
    constexpr int foo = new_sz(); // ok: foo is a constant expression
    ```

- 调试帮助

  - `assert`预处理宏

    ```cpp
    assert(word.size() > threshold);
    ```

  - `NDEBUG` 预处理变量

    assert的行为依赖于一个名为`NDEBUG`的预处理变量的状态。如果定义了`NDEBUG`，则assert什么也不做。

    默认状态下没有定义NDEBUG，此时assert将执行运行时检查。

    ```cpp
    $ CC -D NDEBUG main.C # use /D with the Microsoft compiler
    ```

    这条命令作用等价于在main.c中一开始写了`#define NDEBUG`

    除了用于`assert`外，也可以使用`NDEBUG`编写自己的条件调试代码。如果NDEBUG未定义，将执行`#ifndef`和`#endif`间的代码。如果定义了NDEBUG，将忽略执行这些代码:

    ```cpp
    void print(const int ia[], size_t size)
    {
    #ifndef NDEBUG
        // _ _func_ _ is a local static defined by the compiler that holds the function's name
        cerr << _ _func_ _ << ": array size is " << size << endl;
    #endif
    // ...
    ```

    这段代码中，我们使用`__func__`输出当前调试的函数的名字。编译器为每个函数都定义了`__func__`，它是一个`const char`的静态数组，用于存放函数的名字。

    除了定义的`__func__`之外，预处理器还定义了另外4个对于程序调试很有用的名字:

    ![image-20211128162639154](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211128162639154.png)

#### 函数指针

函数指针指向函数而非对象。函数的类型由它的返回类型和形参类型共同决定，与函数名无关。

```cpp
bool lengthCompare(const string &, const string &);
```

函数类型就是`bool(const string&, const string&)`。

为了声明一个能指向该函数的指针，我们只需用指针替代函数名即可:

```cpp
// pf points to a function returning bool that takes two const string references
bool (*pf)(const string &, const string &); // uninitialized
```

注意：`*pf`两端的括号必不可少。如果不写这对括号，则`pf`是一个返回值为bool指针的函数:

```cpp
// declares a function named pf that returns a bool*
bool *pf(const string &, const string &);
```

当我们把函数名作为一个值使用时，该函数自动转换为指针。如，按照如下形式我们可以讲`lengthCompare`的地址赋值给pf:

```cpp
pf = lengthCompare; // pf now points to the function named lengthCompare
pf = &lengthCompare; // equivalent assignment: address-of operator is optional
```

我们直接使用指向函数的指针调用该函数，无需提前解引用指针:

```cpp
bool b1 = pf("hello", "goodbye"); // calls lengthCompare
bool b2 = (*pf)("hello", "goodbye"); // 等价调用
bool b3 = lengthCompare("hello", "goodbye"); // 等价调用
```

- **函数指针形参**

  形参看起来是函数类型，实际上却是当成指针使用:

  ```cpp
  // 第三个参数是函数类型，它会自动地转换成指向函数的指针
  void useBigger(const string &s1, const string &s2,bool pf(const string &, const string &));
  
  // 等价声明：显式地将形参定义成指向函数的指针
  void useBigger(const string &s1, const string &s2,bool (*pf)(const string &, const string &));
  
  使用示例:
  useBigger(s1, s2, lengthCompare);
  ```

  从上面我们可以发现，直接使用函数指针类型显得很冗长繁琐 类型别名 和 decltype 能让我们简化使用了函数指针的代码:

  ```cpp
  // Func and Func2 都是函数类型
  typedef bool Func(const string&, const string&);
  typedef decltype(lengthCompare) Func2; // equivalent type
  
  // FuncP and FuncP2 是指向函数的指针
  typedef bool(*FuncP)(const string&, const string&);
  typedef decltype(lengthCompare) *FuncP2; // equivalent type
  ```

  则我们上面的`userBigger`重新声明如下:

  ```cpp
  // useBigger 的等价声明，其中使用了类型别名
  void useBigger(const string&, const string&, Func);
  void useBigger(const string&, const string&, FuncP2);
  ```

- **返回指向函数的指针**

  虽然不能返回一个函数，但是能返回指向函数类型的指针。

  ```cpp
  using F = int(int*, int); // F 是一种函数类型，但是不是一个指针类型
  using PF = int(*)(int*, int); // PF 是指针类型
  
  必须显示地将返回类型指定为指针:
  PF f1(int); // 正确：PF是指向函数的指针，f1返回指向函数的指针
  F f1(int); // 错误：F是函数类型，f1不能返回一个函数
  F *f1(int); // 正确： 显示地指定返回类型是指向函数的指针
  ```

  