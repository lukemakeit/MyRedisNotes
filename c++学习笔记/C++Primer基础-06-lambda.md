### lambda

`lambda`表示一个可调用的代码单元。我们可将其理解为一个未命名的内联函数。
lambda和其他任何函数类似，一个lambda具有一个返回类型、一个参数列表和一个函数体。

```cpp
[capture list] (parameter list) -> return type { function body }
```

- `capture_list(捕获列表)`是一个lambda所在函数中定义的局部变量的列表(通常为空);
- `return type`、`parameter list`和`function body`与任何普通函数一样。
- 我们可以忽略参数列表和返回类型，但必须永远包含捕获列表和函数体:
  `auto f = [] {return 42};`

#### 向lambda传递参数

- `lambda`不能有默认参数;

下面是一个与`isShorter`函数完成同样功能的`lambda`:

```cpp
[](const string &a, const string &b) -> bool
{ return a.size() < b.size();}

或者

[](const string &a, const string &b)
{ return a.size() < b.size();}
```

此时我们上面的`stable_sort`既可以写成:

```cpp
// 按长度排序，长度相同的单词维持字典序
stable_sort(words.begin(), words.end(),
    [](const string &a, const string &b)
    { return a.size() < b.size();});
```

**使用捕获列表**
虽然一个`lambda`可以出现在一个函数中，使用其局部变量，但是它只能使用那些明确指明的变量。
捕获列表 指引`lambda`在其内部包含访问局部变量所需的信息。

特别注意:`lambda`捕获列表只用于局部非`static`变量，`lambda`可以直接使用局部`static`变量和在它所在函数之外声明的名字。

问题: 编写一个可以传递给 find_if 的可调用表达式，我们希望这个表达式能将输入序列中每个string的长度与sz参数的值进行比较。

```cpp
void lambdaTest01(){
    vector<string> v1{"hello","zan","a","cao","ab","bbga","good"};
    vector<string>::size_type sz=3;
    auto wc= find_if(v1.begin(),v1.end(),[sz] (const string &a){
        return a.size() > sz;
    });
    if (wc != v1.end()){
        cout <<"Yes,there exists item that the size larger than "<< sz<< endl;
    }else{
        cout <<"Yes,there no exists item that the size larger than "<< sz<< endl;
    }
}
```

`find_if`的调用返回一个迭代器，指向第一个长度不小于捕获参数`sz`的元素。如果这样的元素不存在，则返回`v1.end()`的一个拷贝。

`count_if`函数返回一个计数值，表示谓词有多少次为真。

### lambda捕获与返回

**值捕获**
类似于参数传递，变量的捕获方式也可以是值或者引用。
不过与参数不同，被捕获的变量的值在lambda创建时拷贝，而不是调用时拷贝:

```cpp
void fcn1()
{
    size_t v1 = 42; // 局部变量
    // 将v1拷贝到名为f的可调用对象
    auto f = [v1] { return v1; };
    v1 = 0;
    auto j = f(); // 此时j为42，而不是0。f保存了我们创建它时v1的拷贝
}
```

**引用捕获**

```cpp
void fcn2()
{
    size_t v1 = 42; // local variable
    // f2对象中包含v1的引用
    auto f2 = [&v1] { return v1; };
    v1 = 0;
    auto j = f2(); // j此时是0，f2 保存v1的引用，而不是拷贝
}
```

当然我们也可以从函数中返回一个lambda。函数可以直接返回一个可调用对象，或者返回一个类对象，该类含有可调用对象的数据成员。
如果函数返回一个`lambda`，则与函数不能返回一个局部变量的引用类似，此lambda也不能包含局部变量的的引用捕获。
**可以的话，尽量避免捕获指针或引用，否则程序员应该尽可能保证 引用、指针在调用时是有效的**。

```cpp
void biggies(vector<string> &words,
    vector<string>::size_type sz,
    ostream &os = cout, char c = ' ')
{
    // 与之前的例子一样的重排words的代码
    // 打印count的语句改为打印到os
    for_each(words.begin(), words.end(),
        [&os, c](const string &s) { os << s << c; });
}
```

这里不能拷贝`ostream`对象，因此捕获os的唯一方法就是捕获其引用。

**隐式捕获**
除了显示列出我们希望使用的来自所在函数的变量外，还可以让编译器根据`lambda`体中代码来推断我们要使用哪些变量。
为了指示编译器推断捕获列表，应该在捕获列表中写一个`&`或`=`:

- `&`: 告诉编译器采用捕获引用的方式;
- `=`: 则表示采用值捕获的方式。

```cpp
// sz 为隐式捕获，值捕获方式
wc = find_if(words.begin(), words.end(),
    [=](const string &s)
        { return s.size() >= sz; });
```

部分比那里采用值捕获，其他变量采用引用捕获，可以采用混合使用的方式:

```cpp
void biggies(vector<string> &words,
    vector<string>::size_type sz,
    ostream &os = cout, char c = ' ')
{
    // 其他处理与前例一样
    // os 隐式捕获 ，引用捕获方式；c 显式捕获，值捕获方式
    for_each(words.begin(), words.end(),
        [&, c](const string &s) { os << s << c; });
    // os 显式捕获，引用捕获方式；c 隐式捕获，值捕获方式
    for_each(words.begin(), words.end(),
        [=, &os](const string &s) { os << s << c; });
}
```

混合使用 隐式捕获 和 显式捕获时，捕获列表中的第一个元素必须是`&`或`=`。此符号指定了默认捕获方式。
![30101bdbaed28588428e87dfe0e519da](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/1D226DB6-4D39-4D01-8F85-19DAE61D8D62.png)

#### mutable lambda
**默认情况下，对于一个值被拷贝的变量，lambda不会改变其值**。

**<mark style="color:red;">如果我们希望能改变一个被捕获的变量的值，就必须在参数首加上关键字`mutable`</mark>**。

```cpp
void fcn3()
{
    size_t v1 = 42; // local variable
    // f can change the value of the variables it captures
    auto f = [v1] () mutable { return ++v1; };
    v1 = 0;
    auto j = f(); // j is 43
}
```

**一个 引用捕获 的变量是否可以修改依赖于引用指向的是一个const类型还是一个非const类型**:

```cpp
void fcn4()
{
    size_t v1 = 42; // local variable
    // v1 是一个非const变量的引用
    // 可以通过f2中的引用来改变它
    auto f2 = [&v1] { return ++v1; };
    v1 = 0;
    auto j = f2(); // j is 1
}
```

#### 指定lambda返回类型
截至目前，我们所编写的`lambda`都只包含单一的return语句。
默认情况下，**如果一个lambda体包含return之外的任何语句，则编译器假定次lambda返回void。**
与其他返回void的函数类似，被推断返回 void的lambda不能返回值。

```cpp
transform(vi.begin(), vi.end(), vi.begin(),
    [](int i) { return i < 0 ? -i : i; })
```

函数`transform`接受三个迭代器和一个可调用对象，前两个迭代器表示输入序列，第三个迭代器表示目的位置。
算法对输入序列中每个元素执行可调用对象，并将结果写到目的位置。`tramsform`可以用于写js中的数组的`map`方法哦。

如果我们将上面的代码改写成下面类似等价的`if`语句，则会产生编译错误:

```cpp
transform(vi.begin(), vi.end(), vi.begin(),
    [](int i) { if(i<0) return -i;else return i; })
```

这种情况，怎么改呢？当我们需要为一个`lambda`定义返回类型时，必须使用**尾置返回类型**。

```cpp
transform(vi.begin(), vi.end(), vi.begin(),
    [](int i) -> int
    { if(i<0) return -i;else return i; })
```

#### this指针(class中的lambda)

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

using namespace std;
class Scale
{
public:
    // The constructor.
    explicit Scale(int scale) : _scale(scale) {}
    // Prints the product of each element in a vector object
    // and the scale value to the console.
    void ApplyScale(const vector<int>& v) const
    {
        for_each(v.begin(), v.end(), [=](int n) { cout << n * _scale << endl; });
    }
private:
    int _scale;
};

int main()
{
    vector<int> values;
    values.push_back(1);
    values.push_back(2);
    values.push_back(3);
    values.push_back(4);

    // Create a Scale object that scales elements by 3 and apply
    // it to the vector object. Does not modify the vector.
    Scale s(3);
    s.ApplyScale(values); // 3 6 9 12
}
```

在函数中使用lambda表达式， 可以用显式的this指针:

```cpp
// capture "this" by reference
void ApplyScale(const vector<int>& v) const
{
   for_each(v.begin(), v.end(),
      [this](int n) { cout << n * _scale << endl; });
}

// capture "this" by value (Visual Studio 2017 version 15.3 and later) c++17
void ApplyScale2(const vector<int>& v) const
{
   for_each(v.begin(), v.end(),
      [*this](int n) { cout << n * _scale << endl; });
}
```

也可以用隐式的this指针:

```cpp
void ApplyScale(const vector<int>& v) const
{
   for_each(v.begin(), v.end(),
      [=](int n) { cout << n * _scale << endl; });
}
```

这个Scale类中有一个成员方法`ApplyScale`，使用一个lambda表达式，这个表达式使用了类的数据成员`_scale`。
采用默认值方式`[=]`捕捉变量。
**你可能认为这个lambda表达式也捕捉了`_scale`的一份副本，答案是错的。因为数据成员`_scale`对lambda表达式并不可见，可以用下面的代码验证**:

```cpp
//无法编译，因为_scale并不在lambda捕捉的范围
void ApplyScale(const vector<int>& v) const
{
   for_each(v.begin(), v.end(),
      [_scale](int n) { cout << n * _scale << endl; });
}
```

原来的代码中，每个非静态方法都有一个`this`指针变量,利用隐式的`this`指针，可以接近任何成员变量，所以lambda表达式实际上捕捉的是this指针的副本，所以原来的代码等价于:

```cpp
void ApplyScale(const vector<int>& v) const
{
   for_each(v.begin(), v.end(),
      [this](int n) { cout << n * this->_scale << endl; });
}
```

#### 额外阅读

-  [基础篇：Lambda 表达式和函数对象](https://zhuanlan.zhihu.com/p/143884880)