## C++Primer标准库:泛型算法

都定义在头文件`#include <algorithm>`中。
这些算法一般不直接操作容器，而是遍历由两个迭代器指定的一个元素范围。
一般情况下，这些算法并不直接操作容器，而是遍历由两个迭代器指定的范围来进行操作。

关键概念:
泛型算法不会执行容器的操作，它们只运行于迭代器之上，执行迭代器操作。
算法用于不会执行容器操作：算法可能改变容器中保存的元素的值，也可能在容器内移动元素，但永远不会直接添加或删除元素。

- **find**
```cpp
int val = 42;
auto result = find(vec.cbegin(), vec.cend(), val);
cout << "The value " << val
<< (result == vec.cend()
? " is not present" : " is present") << endl;
```
数组我们也可以使用迭代器:
```cpp
int ia[] = {27, 210, 12, 47, 109, 83};
int val = 83;
int* result = find(begin(ia), end(ia), val);
```

- **count**
```cpp
vector<int> v1{4,5,30,1,3,9,10,3};
auto ret = count(v1.cbegin(),v1.cend(),3);
```

- **accumulate**
`accumulate`的第三个参数类型决定了函数中使用哪个加法运算符以及返回值的类型。
```cpp
vector<int> v1{4,5,30,1,3,9,10,3};
auto sum = accumulate(v1.cbegin(),v1.cend(),0);
cout << "sum="<<sum <<endl;

string sum = accumulate(v.cbegin(),v.cend(),string(""));
string sum = accumulate(v.cbegin(),v.cend(),""); //错误，""类型是const char*,没有+运算符
```

- **equal**
```cpp
equal(roster1.cbegin(),roster1.cend(),roster2.cbegin());
```
a. 使用`==`来比较两个元素;
b. 很重要的前提: 第二个序列至少比第一个序列长;


#### 插入迭代器`back_inserter`
插入迭代器 是一种向容器中添加元素的迭代器。
a. 当我们通过 迭代器 向容器元素赋值时，值被赋予迭代器指向的元素；
b. 当我们通过 插入迭代器 赋值时，一个与赋值号右侧值相等的元素被添加到容器中。

`back_inserter`接受一个 **指向容器的引用**，返回一个与该容器绑定的插入迭代器。
当我们给该迭代器赋值时，赋值运算符会通过`push_back`将一个具有给定值的元素添加到容器中:

```cpp
vector<int> vec;
auto it = back_inserter(vec); // 通过给it赋值，将元素添加到vec中
*it = 42; // vec 中现在有一个元素，值为42
```
我们常常使用`back_inserter`来创建一个迭代器，作为算法的目的位置来使用。如:
```cpp
vector<int> vec;
fill_n(back_inserter(vec), 10, 0); // 添加10个元素到vec中
```

**拷贝算法copy**
此算法接受三个迭代器，前两个表示输入范围，第三个表示目的序列的起始位置。
`copy`返回的是其目的位置迭代器(递增后)的值。

```cpp
int a1[] = {0,1,2,3,4,5,6,7,8,9};
int a2[sizeof(a1)/sizeof(*a1)]; // a2 和 a1 一样大
// ret 指向拷贝到a2的尾元素之后的位置
auto ret = copy(begin(a1), end(a1), a2); // 将a1数据拷贝到a2
```

**替换算法`replace`**
```cpp
// 将所有值为0的元素改为42
replace(ilst.begin(), ilst.end(), 0, 42);
```

如果我们希望保留原序列不变，可以调用`replace_copy`。
```cpp
replace_copy(ilst.cbegin(), ilst.cend(),back_inserter(ivec), 0, 42);
```
此调用后，ilst并未改变，ivec包含ilst的一份拷贝，不过原来在ilst中值为0的元素在ivec中都变成了42。

问题：下面程序有啥问题？
```cpp
vector<int> vec; list<int> lst; int i;
while (cin >> i)
    lst.push_back(i);
copy(lst.cbegin(), lst.cend(), vec.begin());
```
答: **这里`vec`的size()为0，不能用`copy`函数。`copy`函数中必须保证`目标的size() > 源的size()`**。
**所以`vec.begin()`应该改成`back_inserter(vec)`**。

#### 重排容器元素的算法
**消除重复单词**

```c++
void elimDups(vector<string> &words)
{
    // 按字典序排序words，以便查找重复单词
    sort(words.begin(), words.end());
    // unique 重排输入范围，使得每个单词只出现一次
    // 排列在范围的前部，返回指向不重复区域之后一个位置的迭代器
    auto end_unique = unique(words.begin(), words.end());
    // 使用vector操作erase删除重复单词
    words.erase(end_unique, words.end());
}
```
特别注意:**`unique`返回的是一个指向不重复值范围末尾之后的迭代器**。
**其实还有一个`unique_copy`函数，它接受第三个迭代器，表示拷贝不重复元素的目标位置**。后面介绍。

问题： 你认为算法不改变容器大小的原因是什么？
答: 算法中我们只是传入了容器的迭代器，算法并不知道容器具体的大小。
如果我们需要改变容器的大小，只能调用元素本身的算法。

### 定制操作

**谓词(predicate)**
标准库算法所使用的谓词分为两类:
`一元谓词(unary predicate,意味着它们只接收单一参数)`
`二元谓词(binary predicate,意味着它们有两个参数)`

示例:
```cpp
bool isShorter(const string &s1,const string &s2){
    return s1.size() < s2.size();
}
void customSort01(){
    vector<string> v1{"hello","zan","a","cao","ab","bbga","good"};
    sort(v1.begin(),v1.end(),isShorter);
    for(auto &item:v1){
        cout << item <<endl;
    }
}
```
上面这个例子，我们根据字符串长度进行排序。
那如果具有相同长度的元素，我们希望它们的排序顺序保持不变，应当如何处理？
答：可以使用`stable_sort`,维持相等元素的原有顺序。

```cpp
bool isShorter(const string &s1,const string &s2){
    return s1.size() < s2.size();
}
void customSort01(){
    vector<string> v1{"hello","zan","a","cao","ab","bbga","good"};
    // sort(v1.begin(),v1.end(),isShorter);
    stable_sort(v1.begin(),v1.end(),isShorter);
    for(auto &item:v1){
        cout << item <<endl;
    }
}
```

**patition算法**
`partition`，它接受一个谓词，对容器内容进行划分，使用谓词为true的值会排在容器的前半部分，而使谓词为false的值会排在容器的后半部分，算法返回一个迭代器，指向最后一个谓词为true的元素之后的位置。

```cpp
bool isIncludeErr(const string &s1){
    return regex_search(s1,regex("error",regex::icase));
}
void partitionTest01(){
    vector<string> v1{"I am good","Fuck,error happen","There are 3 Errors","niubi",""};
    auto end=partition(v1.begin(),v1.end(),isIncludeErr);
    auto size = end - v1.begin();
    cout << "size::"<< size <<endl;
    for (auto beg=v1.begin();beg != end;++beg){
        cout << *beg << endl;
    }
}
```

#### `lambda`表达式
`lambda`表示一个可调用的代码单元。我们可将其理解为一个未命名的内联函数。
lambda和其他任何函数类似，一个lambda具有一个返回类型、一个参数列表和一个函数体。

```cpp
[capture list] (parameter list) -> return type { function body }
```
- `capture_list(捕获列表)`是一个lambda所在函数中定义的局部变量的列表(通常为空);
- `return type`、`parameter list`和`function body`与任何普通函数一样。
-  我们可以忽略参数列表和返回类型，但必须永远包含捕获列表和函数体:
`auto f = [] {return 42};`

**向lambda传递参数**
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
一个`lambda`通过将局部变量包含在 捕获列表 中来之处将会使用这些变量。
捕获列表 指引`lambda`在其内部包含访问局部变量所需的信息。

特别注意:`lambda`捕获列表只用于局部非`static`变量，`lambda`可以直接使用局部`static`变量和在它所在函数之外声明的名字。

问题: 编写一个可以传递给 find_if 的可调用表达式，我们希望这个表达式能将输入序列中每个string的长度与biggies函数中sz参数的值进行比较。
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

**`for_each算法`**
```cpp
for_each(v1.begin(),end,[](const string &s){
   cout << s << endl;
});
```

完整的`biggies`示例:
```cpp
void biggies(vector<string> &words,
    vector<string>::size_type sz)
{
    elimDups(words); // 将words按照字典序排序，删除重复单词
    // 按长度排序，长度相同的单词维持字典序
    stable_sort(words.begin(), words.end(),
        [](const string &a, const string &b)
        { return a.size() < b.size();});
    // 获取一个迭代器，指向第一个满足 size() >= sz的元素
    auto wc = find_if(words.begin(), words.end(),
        [sz](const string &a)
        { return a.size() >= sz; });
    // 计算满足 size >= sz的元素的数目
    auto count = words.end() - wc;
    cout << count << " " << make_plural(count, "word", "s")
    << " of length " << sz << " or longer" << endl;
    // 打印长度大于等于给定值的单词，每个单词后一个空格
    for_each(wc, words.end(),
        [](const string &s){cout << s << " ";});
    cout << endl;
}
```

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

**可变lambda**
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

**指定lambda返回类型(挺重要的)**
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

#### 参数绑定
对于只在一两个地方使用的简单操作，`lambda`表达式是很有用的。不过我们需要在很多地方使用的操作，通常应该定义一个函数，而不是多次编写相同的`lambda`表达式。
如:

```cpp
bool check_size(const string &s, string::size_type sz)
{
    return s.size() >= sz;
}
```
**这里，我们不能使用函数`check_size`作为`find_if`的参数。因为`find_if`接收一个一元谓词，因此传递给`find_if`的可调用对象必须接受单一参数。所以我们应该怎么解决 sz形参传递问题呢？**

##### 标准库 bind 函数
头文件:`functionnal`,可以讲`bind`函数看做一个通用的函数适配器，它接受一个可调用对象，生成一个新的可调用对象来 适应 源对象的参数列表。
调用`bind`的一般形式是:
`auto newCallable= bind(callable,arg_list)`
- `newCallable`本身是一个可调用对象；
- `arg_list`是一个逗号分割的参数列表,对应给`callable`的参数;
- 当我们调用`newCallable`时，`newCallable`会调用`callable`，并传递给它`arg_list`中的参数;
- `arg_list`中参数可能包含形如`_n`的名字，其中n表示一个整数。这些参数是"占位符"，表示`newCallable`的参数。他们占据了传递给`newCallable`的参数的位置。如，`_1`为`newCallable`的第一个参数，`_2`为第二个参数。

示例:
```c++
using namespace std;
using namespace placeholders; //这个是必须的

bool check_size(const string &s, string::size_type sz)
{
    return s.size() >= sz;
}
// check6是一个可调用对象，接受一个string类型的参数
// 并用词 string 和值 6 来调用check_size
auto check6 = bind(check_size,_1,6);

string s= "hello";
bool b1 = check(s); // check6(s)会调用 check_size(s,6)

// lambda 方式
auto wc = find_if(words.begin(), words.end(), [sz](const string &a))

//bind方式
auto wc = find_if(words.begin(), words.end(),bind(check_size,_1,sz));
```

**使用`placeholders`**
名字`_n`都是定义在`placeholders`命名空间下，如果我们想使用`_n`，必须使用下面的方式:

```cpp
using std::placeholders::_1;
或
using namespace placeholders;
```

**用bind重排参数顺序**

```cpp
// 根据单词长度 由短至长排序
sort(words.begin(), words.end(), isShorter);
// 根据单词长度 由长至短排序
sort(words.begin(), words.end(), bind(isShorter, _2, _1));
```
这样子我们就没必要再写一个`reverse`函数了。

**bind绑定引用参数**
默认情况下，`bind`的那些不是占位符的参数被 **拷贝** 到`bind`返回的可调用对象中。
如:

```cpp
ostream &print(ostream &os, const string &s, char c)
{
    return os << s << c;
}

// 错误: os是不能拷贝的
for_each(words.begin(), words.end(), bind(print, os, _1, ' '));
```
原因在于`bind`拷贝其参数，而我们不能拷贝一个`ostream`。
此时如果我们希望给`bind`传递一个对象引用，而不进行拷贝，就必须使用标准库`ref`函数:

```cpp
for_each(words.begin(), words.end(),
    bind(print, ref(os), _1, ' '));
```
函数`ref`返回一个对象，包含给定的引用，此对象是可以拷贝的。当然标准库中还以一个`cref`函数。

### 再探迭代器
- 插入迭代器(insert iterator): 这些迭代器被绑定到一个容器上，用来想容器插入元素;
- 流迭代器(stream iterator): 这些迭代器被绑定到 输入 或 输出流上，可以用来遍历管理的IO流；
- 反向迭代器(reverse iterator): 这些迭代器向后而不是向前移动。`forward_list`没有反向迭代器;
- 移动迭代器(move iterator): 专用的迭代器而不是拷贝其中元素，而是移动他们。

#### 插入迭代器(insert iterator)
三种类型:
- `back_inserter`: 创建一个使用`push_back`的迭代器;只有支持`push_back`的情况下，才可以使用;
- `front_inserter`: 创建一个使用`push_front`的迭代器;只有支持`push_front`的情况下，才可以使用;
- `inserter`: 创建一个使用insert的迭代器。此函数接收第二个参数，这个参数必须是一个指向给定容器的迭代器。元素将被插入到给定迭代器所表示的元素之前。

当我们调用`inserter(c,iter)`时，我们得到一个迭代器，接下来使用它时，会将元素插入到`iter`原来所指向的的元素之前的位置。
```cpp
*it=val;

其效果和下面代码一样:
it=c.insert(it,val);  // it指向新加入的元素
++it; // 递增it使它指向原来的元素
```
![e3cc743f668ddbe85eebab112835d82e](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/694E0428-0963-4431-8F10-6A72EEBFC857.png)

`front_inserter`生成的迭代器的行为与`inserter`生成的迭代器的完全不一样。当我们使用`front_inserter`时，元素总是插入到容器第一个元素之前。
而`inserter`即使我们传递给它的是首元素，只要我们插入过一次新元素，此时`inserter`不再指向容器首元素了。
```cpp
list<int> 1st = {1,2,3,4};
list<int> lst2, lst3; // 空 lists

// 拷贝完成后，lst2 包含 4 3 2 1
copy(1st.cbegin(), lst.cend(), front_inserter(lst2));
// 拷贝完成后，lst3 包含 1 2 3 4
copy(1st.cbegin(), lst.cend(), inserter(lst3, lst3.begin()));
```
`fron_inserter`生成的迭代器会将插入的元素顺序颠倒过来，而`inserter`和`back_inserter`则不会。

我们来用一下:`unique_copy`函数:
```cpp
vector<string> v1{"hello","a","zan","a","cao","ab","bbga","good","good"};
stable_sort(v1.begin(),v1.end(),[](const string &s1,const string &s2) -> bool 
    { return s1.size() < s2.size(); });

vector<string> destV;
unique_copy(v1.begin(),v1.end(),back_inserter(destV));
for(auto &tmp:destV){
   cout << tmp << endl;
}
```

#### 泛型算法结构

**算法形参模式**

```cpp
alg(beg, end, other args);
alg(beg, end, dest, other args);
alg(beg, end, beg2, other args);
alg(beg, end, beg2, end2, other args);
```
- `dest`一般表示算法可以写入的 目的位置  的迭代器;
- `beg2`和`end2` 表示第二个输入范围;

**算法命名规范**
一些算法使用重载形式传递一个谓词:

```cpp
unique(beg,end); // 使用 == 运算符比较元素
unique(beg,end,comp); // 使用comp比较元素
```
**`_if`版本的算法**
```cpp
find(beg, end, val); // 查找输入范围中val第一次出现的位置
find_if(beg, end, pred); // 查找第一个领pred为真的元素
```

**区分拷贝元素的版本和不拷贝的版本**

```cpp
reverse(beg, end); // 反正输入范围中元素的顺序
reverse_copy(beg, end, dest);// 将元素逆序拷贝到dest

// 从v1中删除奇数元素
remove_if(v1.begin(), v1.end(),
    [](int i) { return i % 2; });
// 将偶数元素从v1版本拷贝到v2；v1不变
remove_copy_if(v1.begin(), v1.end(), back_inserter(v2),
    [](int i) { return i % 2; });
```

类似的还有:
```cpp
replace(beg, end, old_val, new_val);
replace_if(beg, end, pred, new_val);
replace_copy(beg, end, dest, old_val, new_val);
replace_copy_if(beg, end, dest, pred, new_val);
```

### 特定容器算法
![d5cd95ebbe5949bee0706deac70f496e](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/47C45FA8-6E4E-46F9-852D-DCE1C8E446F5.png)
![a0f8184c6a462b8522aa01e734e75a2d](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/7133D2FC-D9B7-480D-925D-C263A16AEDFA.png)
![d3ac9b1fbdfc3bcd5aab41a1c4cc34fb](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/A6258620-987D-48FD-8E83-274918426633.png)
