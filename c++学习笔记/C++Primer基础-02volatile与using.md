## C++Primer基础:volatile、using

### volatile

参考文章: [https://github.com/Light-City/CPlusPlusThings/tree/master/basic_content/volatile](https://github.com/Light-City/CPlusPlusThings/tree/master/basic_content/volatile)

**使用场景**

- `volatile`关键字是一种类型修饰符，普通情况下声明的类型变量表示可以被某些编译器未知的因素(操作系统、硬件、其他线程等)更改。所以使用volatile 告诉编译器不应对这样的对象进行优化;
- `volatile`关键字声明的变量，每次访问时都必须从内存中读取值(没有被`volatile`修饰的变量，可能由于编译器的优化，从CPU寄存器中取值);
- const 可以是 `volatile`(如只读状态的寄存器);
- 指针可以是`volatile`;

**`volatile`应用**

a. 并行设备的硬件寄存器(如状态寄存器)。假设要对一个设备进行初始化，此设备的某一个寄存器为`0xff80000000`

```cpp
int  *output = (unsigned  int *)0xff800000; //定义一个IO端口；  
int   init(void)  
{  
    int i;  
    for(i=0;i< 10;i++)
    {  
    *output = i;  
    }  
}
```

经过编译器优化后，编译器认为前面循环半天都是废话，对最后的结果毫无影响，因为最终只是将output这个指针赋值为 9，所以编译器最后给你编译编译的代码结果相当于:

```cpp
int  init(void)  
{  
    *output = 9;  
}
```

如果你对此外部设备进行初始化的过程是必须是像上面代码一样顺序的对其赋值，显然优化过程并不能达到目的。

**这时候就该使用volatile通知编译器这个变量是一个不稳定的，在遇到此变量时候不要优化**。

`volatile int *output=(unsigned int *)0xff800000;`

b. 一个中断服务子程序中访问变量

```cpp
static int i=0;

int main()
{
    while(1)
    {
    if(i) dosomething();
    }
}

/* Interrupt service routine */
void IRS()
{
	i=1;
}
```

上面示例程序的本意是产生中断时，中断服务子程序IRS响应中断，变更程序变量i，使在main函数中调用dosomething函数。

但是，**由于编译器判断在main函数里面没有修改过i，因此可能只执行一次对从i到某寄存器的读操作，然后每次if判断都只使用这个寄存器里面的“i副本”，导致dosomething永远不会被调用**。

**如果将变量i加上volatile修饰，则编译器保证对变量i的读写操作都不会被优化，从而保证了变量i被外部程序更改后能及时在原程序中得到感知(也就是从内存中读取)**: `volatile static int i=0;`

c. 当多个线程共享某一个变量时，该变量的值会被某一个线程更改，应该用 volatile 声明

作用是**防止编译器优化把变量从内存装入CPU寄存器中，当一个线程更改变量后，未及时同步到其它线程中导致程序出错**。

volatile的意思是让编译器每次操作该变量时一定要从内存中真正取出，而不是使用已经存在寄存器中的值。示例如下：

```cpp
volatile  bool bStop=false;  //bStop 为共享全局变量  
//第一个线程
void threadFunc1()
{
    ...
    while(!bStop){...}
}
//第二个线程终止上面的线程循环
void threadFunc2()
{
    ...
    bStop = true;
}
```

通过第二个线程终止第一个线程循环，如果bStop不使用volatile定义，那么这个循环将是一个死循环，**因为bStop已经读取到了寄存器中，寄存器中bStop的值永远不会变成FALSE**，加上volatile，程序在执行时，每次均从内存中读出bStop的值，就不会死循环了。

**是否了解volatile的应用场景是区分C/C++程序员和嵌入式开发程序员的有效办法**，搞嵌入式的家伙们经常同硬件、中断、RTOS等等打交道，这些都要求用到volatile变量，不懂得volatile将会带来程序设计的灾难。

#### volatile常见问题

(1) 一个参数既可以是const还可以是volatile吗？为什么？

可以。一个例子是只读的状态寄存器。它是volatile因为它可能被意想不到地改变。它是const因为程序不应该试图去修改它。

(2) 一个指针可以是volatile吗？为什么？

可以。尽管这并不常见。一个例子是当一个中断服务子程序修该一个指向一个buffer的指针时。

(3) 下面的函数有什么错误？

```cpp
int square(volatile int *ptr) 
{ 
return *ptr * *ptr; 
} 
```

这段代码有点变态，其目的是用来返回指针ptr指向值的平方，但是，由于ptr指向一个volatile型参数，编译器将产生类似下面的代码：

```cpp
int square(volatile int *ptr) 
{ 
int a,b; 
a = *ptr; 
b = *ptr; 
return a * b; 
} 
```

由于*ptr的值可能被意想不到地改变，因此a和b可能是不同的。结果，这段代码可能返回的不是你所期望的平方值！

正确的代码如下：

```cpp
long square(volatile int *ptr) 
{ 
int a=*ptr; 
return a * a; 
} 
```

**示例**

```cpp
const volatile int local = 10;
int *ptr = (int *)&local;
printf("Initial value of local : %d \n", local);
*ptr = 100;
printf("Modified value of local: %d\n", local); //加上volatile结果为100,不加则为10
```

### using使用

原文链接: [https://github.com/Light-City/CPlusPlusThings/tree/master/basic_content/using](https://github.com/Light-City/CPlusPlusThings/tree/master/basic_content/using)

#### 局部与全局using

```cpp
#include <iostream>
#define isNs1 1
//#define isGlobal 2
using namespace std;
void func() 
{
    cout<<"::func"<<endl;
}
namespace ns1 {
    void func()
    {
        cout<<"ns1::func"<<endl; 
    }
}
namespace ns2 {
#ifdef isNs1 
    using ns1::func;    /// ns1中的函数
#elif isGlobal
    using ::func; /// 全局中的函数
#else
    void func() 
    {
        cout<<"other::func"<<endl; 
    }
#endif
}

int main() 
{
    /**
     * 这就是为什么在c++中使用了cmath而不是math.h头文件
     */
    ns2::func(); // 会根据当前环境定义宏的不同来调用不同命名空间下的func()函数
    return 0;
}
```

还有一种场景的使用:

```cpp
# util.h 中
namespace rs{

#if __has_include(<optional>)
#include <optional>
using std::nullopt;
#elif __has_include(<experimental/optional>)
#include <experimental/optional>
using std::experimental::nullopt;
#else
#error "no available optional headfile"
#endif

}

#tendis_link.cc
namespace rs{
 /**
 * @brief impl NodeCache interface
 * @return return std::nullopt means
 */
	auto TendisLink::GetNodeCache() -> optional<std::reference_wrapper<NodeCache>> {
  	return nullopt;
	}
}
```

#### 取代typedef

C中常用typedef A B这样的语法，将B定义为A类型，也就是给A类型一个别名B。

对应typedef A B，使用using B=A可以进行同样的操作。

```cpp
typedef vector<int> V1; 
using V2 = vector<int>;
```

#### 类中改变访问特性

```cpp
class Base{
public:
 std::size_t size() const { return n;  }
protected:
 std::size_t n;
};

class Derived : private Base {
public:
 using Base::size;
protected:
 using Base::n;
};
```

类Derived private继承了Base，对于它来说成员变量n和成员函数size都是私有的，如果使用了using语句，可以改变他们的可访问性，如上述例子中，size可以按public的权限访问，n可以按protected的权限访问。 