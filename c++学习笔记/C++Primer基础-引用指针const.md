## C++Primer基础:引用、指针、const

### **引用**

引用(reference)为对象起了另一个名字，引用类型引用(refers to)另外一种类型。

- 引用必须初始化;

- 引用的类型要和绑定对象类型严格匹配;

- 引用只能绑定在对象上，而不能与字面值或者某个表达式的计算结果绑定在一起;

- **无法将引用重新绑定到另一个对象上，所以引用必须初始化;**

```cpp
int ival=1024;
int &refVal=ival; //refVal指向ival(是ival的另一个名字)
int &refVal2; //报错: 引用必须被初始化

refVal=2; // 把2赋值给refVal指向的对象
int ii=refVal; //与i=ival执行结果一样

int &refVal4=10; //错误: 引用类型的初始值必须是一个对象,不能与字面值绑定在一起
double dval=3.14;
int &refVal5=dval; //错误: 引用类型初始值必须是int型对象
```

- 为引用赋值: 实际上是将值赋给了引用绑定的对象;
  
  ```cpp
  refVal=2; 把2赋给refVal指向的对象
  int ii=refVal; 与ii=ival执行结果一样
  ```

- 获取引用的值: 实际上是获取了与引用绑定的值的对象
  
  ```cpp
  int &refVal3=refVal; //refVal3绑定到 refVal绑定的队形上，也就是ival上
  int i=refVal; // i被初始化为ival的值
  ```

### **指针**

- 指针本身就是一个对象，允许对指针赋值与拷贝; 而且在指针的生命周期内可以先后指向几个不同的对象;

- 指针无需在定义时初始化。**在块作用域内定义的指针如果没有初始化，也将拥有一个不确定值;**

- 因为引用不是对象，没有实际地址，所以不能定义指向引用的指针;

```cpp
int ival=42;
int *p=&ival; // p存放变量ival的地址 或者说p是指向变量ival的指针
```

**空指针**

```cpp
int *p1=nullptr; //等价于int *p1=0;
int *p2=0;

//需要首选#include cstdlib
int *p3=NULL;

int zero=0;
pi=zero; //错误: 不能把int变量直接复制给指针，尽管该变量值是0也不行
```

**指针操作**

- 如果指针值是0，条件取false;

- 如果指针值非0，条件取true;

```cpp
int ival=1024;
int *pi=0; //pi合法，是一个空指针
int *pi2=&ival; //pi2是一个合法指针，存放ival地址

if(pi) // false
...

if(pi2) //pi2指向ival,因此他的值不是0，条件值是true
```

**指向指针的引用**

引用本身不是一个对象，因此不能定义指向引用的指针。

但是指针是对象，所以存在对指针的引用。

```cpp
int i=42;
int *p;
int *&r=p; // r是一个对指针p的引用

r=&i; // r引用了一个指针，此时r赋值&i就是令p指向i
*r=0; // 解引用r得到i,也就是p指向的对象，将i的值改为0
```

tips: 理解r到底是什么类型，最简单的方法从离变量名最近的符号(`&r`的符号是`&`)开始看起，此时我们可以发现r是一个引用。

声明符其余部分用以确定r引用的类型是什么，此例中符号`*`说明r引用的是一个指针。最后，声明的基本数据类型部分支出r引用的是一个int指针。

### **const**

```cpp
const int bufSize=512;
```

- 编译器将在编译过程中把用到该变量的地方都替换成对应的值。也就是说，编译器会找到代码中所有用到buffSize的地方，然后用512替换;

- 默认情况下，**const变量被设定为仅在当期那文件内有效**。当多个文件中出现同名的`const`变量时，其实等同于在不同文件中分别定义了独立的变量。

- 如果确实需要在多个文件间共享某个变量，则对const变量**不管是声明还是定义都添加`extern`关键字**，这样定义一次即可:

```cpp
/*file_1.cc 定义并初始化一个变量，该变量能被其他文件访问*/
extern const int bufSize=fcn();

/* file_1.h 头文件 */
extern const int bufSize; //与file_1.cc中定义的bufSize是同一个
```

**const的引用**

把引用绑定到const对象上，则称为**对常量的引用（reference to const）**。

**对常量的引用不能被用作修改它所绑定的对象:**

```cpp
const int ci=1024;
const int &r1=c1; //正确: 引用及其对应对象都是常量
r1=42; //错误: r1是对常量的引用
int &r2=ci; //错误: 试图让一个非常量的引用指向一个常量对象
```

**初始化和对const的引用**

引用的类型必须与其所引用对象的类型一致。但是有两个例外:

- 第一个: 在初始化常量引用时允许用任意表达式作为初始值，只要该表达式的结果能转换成引用的类型即可。尤其**允许为一个常量引用绑定一个非常量的对象、字面值等**:

```cpp
int i=42;
const int &r1=i; 允许将const int&绑定到一个普通的int对象上
const int &r2=42; //正确:r2是一个常量引用,绑定了一个字面值
const int &r3=r1*2; //正确: r3是一个常量引用
int &r4=r1*2; //错误: r4是非常量引用
```

下面来解释下原因,如下 当一个常量引用绑定到另一中类型时会发生什么?

```cpp
double dval=4.14;
const int &ri=dval;
```

此处ri应用了一个int类型的数，dval却是一个双精度浮点数。此时为了确保让ri绑定到一个整数，编译器会将上面代码变成如下形式:

```cpp
const int temp=dval; //双精度浮点数生成一个临时整型常量
const int &ri=temp; //让ri绑定这个临时常量
```

那为啥`int &ri=dval`这种不行呢？如果这种成立，`ri`指向的是临时常量，还怎么做`dval`的别名。

**const引用可以引用非const对象**

```cpp
int i=42;
int &ri=i; //引用ri绑定对象i
const int &r2=i; //r2也绑定对象i,但不允许通过r2修改i的值
r1=0; //正确
r2=0; //错误
```

允许令一个指向常量的指针指向一个非常量的对象:

```cpp
double dval=3.14; //dval 是一个双精度浮点数，它的值可改变
cptr=&dval; // 正确: 但不能通过cptr改变dval的值
```

**const指针**

**常量指针(const pointer)** 必须初始化，一旦初始化，则它的值不能改变(指针指向哪个地址不能变)。

```cpp
int errNumb=0;
int *const curErr = &errNumb; // curErr将一直指向一个errNumb
const double pi=3.414;
const double *const pip=π //pip是一个指向常量对象的常量指针

*pip=2.72; //错误

const double pi2=3.33;
pip=&pi2; //错误

*curErr=0; //正确
```

### 类型

**定义类型别名(type alias)**

第一种方法:

```cpp
typedef double wages; //wags 是 double的同义词
typedef wages base,*p; //base是一个double同义词，p是double *的同义词
```

新方法:

```cpp
using SI=Sales_item; //SI是Sables_item的同义词
```

**auto类型说明符**

```cpp
auto item=val1+val2;// 有编译器根据val1 和 val2相加的结果自动推断出item的类型
auto i=0,*p=&i; //正确: i是整数，p是整数指针
auto sz=0, pi=3.14; //错误，sz 和 pi的类型不一致
```

**decltype类型指示符**

这种情况下使用: **希望从表达式的类型推断出要定的变量的类型，但不想用该表达式的值初始化变量。**

```cpp
decltype(f()) sum=x; //sum的类型就是函数f的返回值类型
```

编译器不实际调用函数f，而是使用当调用发生时f的返回值类型作为sum的类型。

```cpp
const int ci=0,&cj=ci;
decltype(ci) x=0; //x的类型是const int
decltype(cj) y=x; //y的类型是const int &,y绑定到变量x
decltype(cj) z; //错误: z是一个引用，必须初始化
```
