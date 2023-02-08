## C++Primer基础: 类

- 类可以包含多个构造函数，和其他重构函数类似。不同构造函数之间必须在参数数量或参数类型上有所区别;

- 合成的默认构造函数

  默认构造函数没有任何实参，由编译器创建:

  - 如果存在类内的初始值，用它来初始化成员;

  - 否则，默认初始化该成员;

  - 某些类不能依赖于合成的默认构造函数:

    - 一旦我们定义了其他构造函数，那么除非我们再定义一个默认的构造函数，否则类将没有默认构造函数；

    - 对于某些类来说，合成的默认构造函数可能执行出错。

      如果定义在块中的内置类型或复合类型(比如数组和指针)的对象被默认初始化，则它们的值将是未定义的。

    - 有时候编译器不能为某些类合成默认构造函数，如，如果类中包含一个其他类类型的成员且和成员的类型没有默认构造函数，那么编译器将无法初始化该成员。

      对于这种类来说，我们必须自定义默认构造函数，否则该类没有可用的默认构造函数。

      ```cpp
      Sales_data() = default;
      Sales_data(const std::string &s): bookNo(s) {}
      Sales_data(const std::string &s,unsigned cnt,double price):bookNo(s),units_sold(cnt),revenue(cnt*price){}
      Sales_data(std::istream &);
      ```

    - `=default`的含义

      通过`Sales_data() = default`我们手动为`Sales_data`类生成了默认构造函数。

#### 友元

类可以允许其他类 或 函数访问它的非public成员，方法是令其他类或者函数成为他的**友元(friend)**。

```cpp
class Sales_data {
friend Sales_data add(const Sales_data &,const Sales_data &);
friend std::ostream &print(std::ostream &,const Sales_data &);
friend std::istream &read(std::istream &,Sales_data &);
...
}

Sales_data add(const Sales_data &,const Sales_data &);
std::ostream &print(std::ostream &,const Sales_data &);
std::istream &read(std::istream &,Sales_data &);
```

- 友元声明只能出现在类定义的内部，但是在类内出现的具体位置不限;

- 友元的声明仅仅指定了访问的权限，而非一个通常意义上的函数声明。

  所以我们必须在友元声明之外再专门对函数进行一次声明。

- 通常情况，友元的声明 和 类本身放置在一个头文件中；

#### 类的其他特性

- 定义一个类型成员

  Screen表示显示器中一个窗口。每个Screen包含一个用于保存Screen内容的string成员和`cursor`、`height`、`width`三个成员，分别表示光标位置以及屏幕的高和宽。

  ```cpp
  class Screen {
  public:
      typedef std::string::size_type pos;
      或者
      using pos = std::string::size_type;
      
      Screen() = default; // 因为Screen有其他构造函数，所以本函数是必需的
      // cursor 被其类内初始值初始化为0
      Screen(pos ht, pos wd, char c): height(ht), width(wd),
      contents(ht * wd, c) { }
      char get() const // 读取光标处的字符
      { return contents[cursor]; } // 隐士内联
      inline char get(pos ht, pos wd) const; // 显示内联
      Screen &move(pos r, pos c); // 之后再内联
  private:
      pos cursor = 0;
      pos height = 0, width = 0;
      std::string contents;
  };
  ```

- 可变数据成员:`mutable`

  一个可变数据成员(mutable data number)永远不会是const，即使它是const对象的成员。

  如下，我们将给Screen添加一个名为`access_ctr`的可变成员，通过它我们可以追踪每个Screen成员函数被调用过多少次。

  ```cpp
  class Screen {
  public:
      void some_member() const;
  private:
      mutable size_t access_ctr; // 即使在一个const对象内也可以被修改
      // other members as before
  };
  void Screen::some_member() const
  {
      ++access_ctr; // 保存一个计数值，用于记录成员函数被调用的次数
      // whatever other work this member needs to do
  }
  ```

- 类数据成员的初始值

  我们继续定义一个窗口管理类并用它表示显示器上的一组Screen。

  同时默认情况下，我们希望`Window_mgr`类开始时总是有默认初始化的Screen。

  在C++11新标准中，最好的方式就是把这个默认值声明成一个类内初始值。

  ```cpp
  class Window_mgr {
  private:
      // Screens this Window_mgr is tracking
      // by default, a Window_mgr has one standard sized blank Screen
      std::vector<Screen> screens{Screen(24, 80, ' ') };
  };
  ```

- 返回`*this`的成员函数

  ```cpp
  class Screen {
  public:
      Screen &set(char);
      Screen &set(pos, pos, char);
      // other members as before
  };
  inline Screen &Screen::set(char c)
  {
      contents[cursor] = c; // set the new value at the current cursor location
      return *this; // return this object as an lvalue
  }
  inline Screen &Screen::set(pos r, pos col, char ch)
  {
      contents[r*width + col] = ch; // set specified location to given value
      return *this; // return this object as an lvalue
  }
  ```

  如上我们就可以像下面方法操作屏幕内容:

  ```cpp
  // move the cursor to a given position, and set that character
  myScreen.move(4,0).set('#');
  ```

#### 友元再探

#### 类之间的友元关系

如我们需要为`Window_mgr`添加一个名为`clear`的成员，它负责把一个指定的Screen的内容都设为空白。

为了完成这一任务，clear需要访问Screen的私有成员。因此Screen必须把Window_mgr指定为它的友元

```cpp
class Screen {
    // Window_mgr members can access the private parts of class Screen
    friend class Window_mgr;
    // ... rest of the Screen class
};

class Window_mgr {
public:
    // location ID for each screen on the window
    using ScreenIndex = std::vector<Screen>::size_type;
    // reset the Screen at the given position to all blanks
    void clear(ScreenIndex);
private:
    std::vector<Screen> screens{Screen(24, 80, ' ')};
};
void Window_mgr::clear(ScreenIndex i)
{
    // s is a reference to the Screen we want to clear
    Screen &s = screens[i];
    // reset the contents of that Screen to all blanks
    s.contents = string(s.height * s.width, ' ');
}
```

- 如果一个类指定了友元类，则友元类的成员函数可以访问此类包括非公有成员在内的所有成员。

- 友元不存在传递性，也就是说`Window_mgr`有它自己的友元，则这些友元并不能理所当然地具有访问Screen的特权。每个类负责控制自己的友元类或者友元函数。

#### 令成员函数作为友元

```cpp
class Screen {
    // Window_mgr::clear must have been declared before class Screen
    friend void Window_mgr::clear(ScreenIndex);
    // ... rest of the Screen class
};
```

如果想令某个成员函数作为友元，我们必须仔细组织程序结构以满足声明 定义的彼此依赖关系:

- 首先定义`Window_mgr`类，其中声明clear函数，但不能定义它。在clear使用Screen的成员之前必须先声明Screen;

- 接下来定义Screen，包括对于clear的友元声明；

- 最后定义clear，此时它才可以使用Screen的成员。

#### 类的作用域

- 作用域和定义在类外部的成员

  一旦遇到类名，定义的剩余部门就是在类的作用域中了。这里的剩余部分包括参数列表和函数体。

  ![image-20211128213322443](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211128213322443.png)

- 另一方面，函数的返回类型通常出现在函数名之前。

  因此当成员函数定义在类的外部时，返回类型中使用的名字都位于类的作用域之外。

  因此返回类型必须指明它是哪个类的成员:

  ```cpp
  class Window_mgr {
  public:
      // add a Screen to the window and returns its index
      ScreenIndex addScreen(const Screen&);
      // other members as before
  };
  
  // return type is seen before we're in the scope of Window_mgr
  Window_mgr::ScreenIndex
  Window_mgr::addScreen(const Screen &s)
  {
      screens.push_back(s);
      return screens.size() - 1;
  }
  ```

  ![image-20211128213544067](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211128213544067.png)

### 构造函数再探

- 构造函数初始值列表

  下面这种写法并不好，先初始化再赋值:

  ```cpp
  // legal but sloppier way to write the Sales_data constructor: no constructor initializers
  Sales_data::Sales_data(const string &s,
      unsigned cnt, double price)
  {
      bookNo = s;
      units_sold = cnt;
      revenue = cnt * price;
  }
  ```

- 构造函数和初始值有时必不可少

  - 有时我们可以忽略数据成员初始化和赋值之间的差异，但并非总能这样；

  - **如果成员是const或者引用的话，必须将其初始化；**

  - **如果成员是某种类型且没有默认构造函数时，也必须将这个成员初始化**;

    ```cpp
    示例一:
    class ConstRef {
    public:
        ConstRef(int ii);
    private:
        int i;
        const int ci;
        int &ri;
    };
    下面这种写法就是错误的: ci、ri必须初始化
    ConstRef::ConstRef(int ii)
    { // assignments:
        i = ii; // ok
        ci = ii; // error: cannot assign to a const
        ri = i; // error: ri was never initialized
    }
    正确的写法:
    ConstRef::ConstRef(int ii): i(ii), ci(ii), ri(i) { }
    ```

- 成员初始化的顺序

  - 成员初始化的顺序与他们在类中定义出现顺序一致；

  - 第一个成员先被初始化，然而第二个；

  - 构造函数初始值列表中初始值的前后位置关系不会影响实际的初始化顺序。

  - 一般来说，初始化顺序设计没什么特别要求。不过如果一个成员是用另一个成员来初始化，则两个成员的初始化顺序很关键。

    ```cpp
    class X {
        int i;
        int j;
    public:
        // undefined: i is initialized before j
        X(int val): j(val), i(j) { }
    };
    ```

    这个例子中，从构造函数初始值形式来看仿佛先用val初始化j，然后用j初始化i。实际上，i先初始化，此时j是未定义的0值。替换写法:

    ```cpp
    X(int val): i(val), j(val) { }
    ```

    