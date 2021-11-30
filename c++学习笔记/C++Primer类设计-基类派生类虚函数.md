## C++ Primer 类设计: 基类、派生类、虚函数

- `Quote`基类，表示按照原价销售书籍;
- `Bulk_quote`，表示打折销售书籍;
- 如果某些函数，基类希望它的派生类各自定义合适自身的版本，此时基类就将这些函数声明成**虚函数(virtual function)**;
- 派生类必须在其内部**对所有重新定义的虚函数进行声明**，派生类可以在这样的函数之前加上`virtual`关键字，不过这不是必须的；(派生类不一定要重写 虚函数哦，只是说 重写时需要重新声明)
- C++11新标准允许派生类显式地注明它将使用哪个成员函数改写基类的虚函数，在该函数的形参列表后增加`override`关键字即可。
- 继承关系中，根节点的类通常都会定义一个虚析构函数(也就是 析构函数 也定义为 虚函数);

**成员函数和继承**
- 派生类可以继承基类的成员，然而当遇到`net_price`这样的类型相关的操作时，派生类必须对其重新定义;
- 关键字`virtual`只能出现在 类内部的声明语句之前 而不能用于 类外部的函数定义;
- 如果基类把一个函数声明为虚函数，则该函数在派生类中 隐式地也是虚函数；
- 成员函数如果没有被声明为 虚函数，则其解析过程发生在 编译时 而不是运行时；

**访问控制与继承**

- 派生类能访问基类的`public`、`protected`的成员；不能访问`private`成员;

#### 定义派生类
派生类列表: 首先是一个冒号，然后紧跟以逗号分隔的基类列表，其中每个基类前面可以有访问说明符:`public`、`protected`、`private`。
这里的访问说明符作用是： 控制派生类从基类继承而来的成员是否对派生类的用户可见。
如果一个派生是`public`的，则基类的公有成员也是派生类接口的组成部分。此外，我们能将公有派生类型的对象绑定到基类的引用 和指针上。
(大部分应该都用`public`派生)
如:`class Bulk_quote: public Quote`

**派生类中的虚函数**
- **派生类经常(但并不总是)覆盖它继承的虚函数。如果派生类没有覆盖基类中的某个虚函数，则该虚函数的行为类似其他普通函数，派生类会直接继承其在基类中的版本**;
- 派生类可以在覆盖函数时使用`virtual`关键字，也可以不用;
- 派生类覆盖基类某函数时，建议加上`override`关键字;

**派生类构造函数**

- 尽管派生类对象中含有从基类继承而来的成员，但是派生类并不能直接初始化这些成员;
- 和其他创建基类对象的代码一样，派生类也必须 **使用基类的构造函数来初始化它的基类部分**
```cpp
Bulk_quote(const std::string &book,double p,std::size_t qty,double disc):
Quote(book,p),min_qty(qty),discount(disc)
{
}
```

**派生类使用基类成员**
派生类可以访问基类的`public`和`protected`成员:

```cpp
// price是父类成员
double Bulk_quote::net_price(std::size_t cnt) const{
    if(cnt > min_qty){
        return cnt * (1 - discount) * price;
    }else{
        return cnt * price;
    }
}
```

**继承与静态成员**
如果基类定义了一个静态成员，则在整个继承体系中只存在该成员的唯一定义。
不论从基类中派生出多少个派生类，对于每个静态成员来说都只存在唯一实例。

```cpp
class Base {
public:
    static void statmem();
};
class Derived : public Base {
    void f(const Derived&);
};
void Derived::f(const Derived &derived_obj)
{
    Base::statmem(); // ok: Base定义了 statemem
    Derived::statmem(); // ok: Derived 继承了 statmem
    // ok: 派生类的对象能访问基类的静态成员
    derived_obj.statmem(); // 通过 Derived 对象访问
    statmem(); // 通过this对象访问
}
```

**派生类的声明**
派生类的声明与其他类声明一样，**声明中包含类名但是不包含它的派生列表**:

```cpp
class Bulk_quote : public Quote; // 错误: 派生列表不能出现在这里
class Bulk_quote; // 正确: 声明派生类的正确方式
```

**被用作基类的类**
如果我们想将某个类用作基类，则**该类必须已经定义而非仅仅声明**

```cpp
class Quote; //声明但未定义
// 错误: Quote必须被定义
class Bulk_quote: public Quote {...};
```
这一规定的原因很明显：派生类中包含并且可以使用它从基类继承而来的成员，为了使用这些成员，派生类当然要知道它们是什么。

**防止继承的发生**
如果我们不想其他类继承某个类，`C++11`新标准提供了一种防止继承发生的方法: 在类名跟一个关键字`final`

```cpp
class NoDerived final { /* */ }; // NoDerived 不能作为基类
class Base { /* */ };
// Last 是final, 我们不能继承Last
class Last final : Base { /* */ }; // Last 不能作为基类
class Bad : NoDerived { /* */ }; // error: NoDerived is final
class Bad2 : Last { /* */ }; // error: Last is final
```

注意: **和内置指针一样，智能指针类也支持派生类向基类类型的转换，意味着我们可以将一个派生类对象的指针存储在基类的智能指针内<mark style="color:red;">智能指针也支持多态</mark>**。

**不存在从基类向派生类的隐式类型转换.....**

```cpp
Quote base;
Bulk_quote* bulkP = &base; // 错误: 不能将基类转换成派生类
Bulk_quote& bulkRef = base; //错误: 不能将基类转换成派生类
```
**即使基类指针 或 引用绑定在一个派生类对象上，我们也不能执行从基类向派生类的转换**

```cpp
Bulk_quote bulk;
Quote *itemP = &bulk; // 正确: 动态类型是Bulk_quote
Bulk_quote *bulkP = itemP; //错误: 不能将基类转换成派生类
```
- 如果在基类中有一个 或 多个虚函数，我们可以使用`dynamic_cast`请求一个类型转换，**该类型转换的安全检查将在运行时执行**;
`dynamic_cast1`可以获取目标对象的引用或指针：
```cpp
T1 obj;
T2* pObj = dynamic_cast<T2*>(&obj);//转换为T2指针，失败返回NULL
T2& refObj = dynamic_cast<T2&>(obj);//转换为T2引用，失败抛出bad_cast异常
```
- 如果我们已知某个基类向派生类的转换是安全的，则我们可以使用`static_cast`来强制覆盖掉编译器的检查工作;
[https://www.cnblogs.com/QG-whz/p/4509710.html](https://www.cnblogs.com/QG-whz/p/4509710.html)

### 虚函数
在C++语言中，当我们使用基类的引用 或 指针调用虚成员函数时会执行动态绑定。
动态绑定必须满足两个条件:

- **<mark style="color:red;">只有虚函数才能进行动态绑定，非虚函数不进行动态绑定</mark>**；
- **<mark style="color:red;">必须通过基类类型的引用或指针进行函数调用</mark>**;

**final和override说明符**
派生类如果定义了一个函数与基类中虚函数的名字相同，但是形参列表不同，这仍然是合法的行为。
编译器认为新定义的这个函数与基类中原有的函数是 **相互独立的**。这时派生类的函数并没有覆盖掉基类中的版本，这种是很可能发生的错误。

针对这种情况，新标准中我们可以使用`override`关键字说明派生类中的虚函数。
**如果`override`标记了某个函数，但该函数并没有覆盖已存在的函数，此时编译器将报错**:

```cpp
struct B {
    virtual void f1(int) const;
    virtual void f2();
    void f3();
};
struct D1 : B {
    void f1(int) const override; // ok: f1 与 基类中 f1匹配
    void f2(int) override; // error: B没有形如 f2(int)的函数
    void f3() override; // error: f3 不是虚函数
    void f4() override; // error: B 没有名为f4的函数
};
```
我们还能把某个函数指定为`final`,如果我们已经把函数定义成`final`。则之后任何尝试覆盖该函数的操作都将引发错误:
```cpp
struct D2 : B {
    // 从B继承 f2() and f3() from B，覆盖f1(int)
    void f1(int) const final; // 不允许后续的其他类覆盖 f1(int)
};
struct D3 : D2 {
    void f2(); // ok: 覆盖从间接基类B继承而来的 f2
    void f1(int) const; // error: D2已经将f2声明成final
};
```

**虚函数与默认实参**
如果虚函数使用默认实参，则基类和派生类中定义的默认实参最好一致。

**回避虚函数的机制**

```cpp
double undiscounted = baseP->Quote::net_price(42);
```