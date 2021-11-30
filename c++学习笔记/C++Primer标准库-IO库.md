## C++Primer 标准库: IO库

IO库设施:

- `istream(输入流)类型`: 提供输入操作;

- `ostream(输出流类型)`: 提供输出操作;

- `cin`: 一个istream对象，从标准输入读取数据;

- `cout`: 一个ostream对象，通常用于向标准输出中输出信息;

- `>>运算符`，从一个istream对象中读取数据;

- `<<运算符`，想一个ostream对象写入输出数据;

- `getline函数`:从一个给定的istream读取一行数据(不含换行符)，存入一个给定的string对象中;

#### IO类

`iostream`定义了用于读写流的基本类型;

`fstream`:定义了读写命名文件的类型;

`sstream`:定义了读写内存string对象的类型;

![image-20211128230225506](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211128230225506.png)

##### IO 对象无拷贝或赋值

```cpp
ofstream out1, out2;
out1 = out2; // error: 不能对流对象赋值
ofstream print(ofstream); // error: 不能初始化ofstream参数
out2 = print(out2); // error: 不能拷贝流对象
```

因为不能拷贝IO对象，因此我们不能将形参或者返回类型设置为流类型。

进行IO操作的函数通常以引用方式传递和返回流。

读写一个IO对象会改变其状态，因此传递和返回引用不能是const。

##### 条件状态

IO操作一个与生俱来的问题是可能发生错误。一些错误是可恢复的，而一些则发生在系统深处，无法恢复。

所以IO类定义了一些函数和标志，可以帮助我们访问和操作流的条件状态:

![image-20211128230807011](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211128230807011.png)

![image-20211128230905171](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211128230905171.png)

确定一个流对象的状态最简单的方法就是将它当做一个条件来用:

```cpp
while(cin >>word)
```

**查询流的状态**

将流作为条件，只能告诉我们流是否有效，无法告诉我们具体发生了什么。

IO定义了一个与机器无关的`iostate`类型。

`badbit`表示系统级错误，如不可恢复的读写错误，通常情况下，一旦`badbit`被置位，流就无法再使用了;

`failbit`一般表示可恢复的错误，如期望读取数值却读取出一个字符等错误；

如到达文件结束，`eofbit`、`failbit`都会被置位；

`goodbit`值为0，表示流未发生错误。

**管理条件状态**

流对象的`rdstate`成员返回一个iostate值，对应流的当前状态。

`setstate`操作将给定条件置位，表示发生了对应错误。

`clear`成员有接收参数版本 也有 不用接受iostate类型参数的版本。

```cpp
// remember the current state of cin
auto old_state = cin.rdstate(); // 记住cin的当前状态
cin.clear(); // 使cin有效
process_input(cin); // 使用cin
cin.setstate(old_state); // 恢复cin原有状态
```

#### 管理输出缓冲

每个输出流都管理一个缓冲区，用于保存程序读写的数据。

因为有了缓冲机制，操作系统就可以讲程序的多个输出操作组合成一个单一的系统级写操作。

导致缓冲刷新的原因很多:

- 程序正常结束，作为main函数的return操作的一部分，缓冲刷新执行；

- 缓冲区满时，需要刷新缓冲，而后新的数据才能继续写入缓冲区；

- 我们可以操作如`endl`来显示刷新缓冲区；

- 每次输出操作后，我们可以用操作符`unitbuf`设置流的内部状态，来清空缓冲区。默认情况下，对`cerr`是设置`unitbuf`的，因此cerr的内容是立即刷新的；

- 一个输出流可能被关联到另一个流。在这种情况下，当读写被关联的流时，关联到的流的缓冲区立即刷新。

  如，默认情况下，cin和cerr都关联到cout，因此读cin或写cerr都会导致cout的缓冲区刷新。

**刷新输出缓冲区**

```cpp
cout << "hi!" << endl; // 输出hi和一个换行符，然后刷新缓冲区
cout << "hi!" << flush; // 输出hi，然后刷新缓冲区，不附加任何额外字符
cout << "hi!" << ends; // 输出hi和一个空字符，然后刷新缓冲区
```

**unitbuf操作符**

```cpp
cout << unitbuf; // 所有输出操作后都会立即刷新缓冲区
cout << nounitbuf; //回到正常的缓冲方式
```

注意: 如果程序异常终止，输出缓冲区是不会被刷新的。当一个程序崩溃后，它所输出的数据很可能停留在输出缓冲区中等待打印。

#### 文件输入和输出

![image-20211128231731015](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211128231731015.png)

**使用fstream代替`iostream &`**

```cpp
ifstream input(argv[1]); // 打开销售记录文件
ofstream output(argv[2]); // 打开输出文件
Sales_data total; // 保存销售总额的变量
if (read(input, total)) { // 读取销售总额的变量
    Sales_data trans; // 保存下一条销售记录的变量
    while(read(input, trans)) { // 读取剩余的记录
        if (total.isbn() == trans.isbn()) // 检查isbn
            total.combine(trans); // 更新销售总额
        else {
            print(output, total) << endl; // 打印结果
            total = trans; // 处理下一本书
        }
    }
    print(output, total) << endl; // 打印最后一本书的销售额
} else // 文件中无输入数据
    cerr << "No data?!" << endl;
```

**成员函数open和close**

如果我们定义了一个空文件流对象，可以随后调用open来将它与文件关联起来:

```cpp
ifstream in(ifile); // 构建一个ifstream并打开给文件
ofstream out; // 输出文件流未与任何文件相关联
out.open(ifile + ".copy"); // 打开指定文件
if(out) //判断打开是否成功
```

将文件流关联到另一个文件，必须首先关闭已经关联的文件。一旦文件成功关闭，我们可以打开新的文件:

```cpp
in.close(); //关闭文件
in.open(ifile+"2"); //打开另一个文件
```

**自动构造好析构**

```cpp
// 对每个传递给程序的文件执行循环操作
for (auto p = argv + 1; p != argv + argc; ++p) {
    ifstream input(*p); // 创建输出流并打开文件
    if (input) { // 如果文件打开成功，处理 此文件
        process(input);
    } else
        cerr << "couldn't open: " + string(*p);
} // 每个循环步 input 都会离开次作用域，因此会被销毁
```

因为`input`是`while`循环的局部变量，它在每个循环步中都要创建和销毁一次。当`fstream`对象离开其作用域时，与之关联的文件会自动关闭。下一步循环中，input会再次被创建。

**当一个fstream对象被销毁时，close会自动调用。**

**文件模式**

每个流都有一个关联的文件模式(file mode),用来指出如何使用文件。

![image-20211129102035512](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129102035512.png)

一些限制:

- 只可以对`ofstream`或`fstream`对象设置`out`模式;

- 只可以对`ifstream`或`fstream`对象设置`in`模式;

- 只有当`out`也被设定时才可以设定`trunc`模式;

- 只要`trunc`没设定，就可以设定app模式。app模式下，即使没有显示指定out模式，文件总是以输出方式打开。

- 默认情况下，即使我们没有指定`trunc`，以out模式打开的文件也会被截断。为了保留out模式打开的文件的内容，我们必须同时指定`app`模式，这样只会讲数据追加写到文件末尾；或者同时指定`in`模式，即打开文件同时进行读写操作。

- `ate`和`binary`模式可以用于任何类型的文件流对象，且可以与其他任何文件模式组合使用。

与`ifstream`关联的文件默认是以`in`模式打开；与`ofstream`关联的文件默认以out模式打开；与`fstream`关联的文件默认以`in`或`out`模式打开。

**以out模式打开文件丢弃已有数据**

```cpp
// 在这几条语句中，file1都被截断
ofstream out("file1"); // 隐含以输出模式打开文件并截断文件
ofstream out2("file1", ofstream::out); // 隐含地截断文件
ofstream out3("file1", ofstream::out | ofstream::trunc);
// 为了保留文件内容，我们必须显示指定app模式
ofstream app("file2", ofstream::app); // 隐含输出模式
ofstream app2("file2", ofstream::out | ofstream::app);
```

保留被`ofstream`打开的文件中已有数据的唯一方法是显式指定`app`或`in`模式。

**每次调用open时都会确定文件模式**

```cpp
ofstream out; // 未指定文件打开模式
out.open("scratchpad"); // 模式隐含设置为输出和截断
out.close(); // 关闭out，以便我们将其用于其他文件
out.open("precious", ofstream::app); // 模式为输出和追加
out.close();
```

#### String流

头文件`sstream`。

![image-20211129102505244](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129102505244.png)

##### 使用istringstream

当我们的某些工作是对整行文本进行处理，而其他一些工作是处理行内的单个单词时，通常可以使用`istringstream`。

如下面这个文件:

```cpp
morgan 2015552368 8625550123
drew 9735550130
lee 6095550132 2015550175 8005550000
```

此时:

```cpp
// 成员默认是共有
struct PersonInfo {
    string name;
    vector<string> phones;
};
string line, word; // 分别保存来自输入的一行和单词
vector<PersonInfo> people; // 保存来自输入的所有记录
// 逐行从输入中读取数据，直至cin遇到文件末尾
while (getline(cin, line)) {
    PersonInfo info; // 创建一个保存记录数据的对象
    istringstream record(line); // 将记录绑定到刚读入的行
    record >> info.name; // 读取名字
    while (record >> word) // 读取电话号码
        info.phones.push_back(word); // 保持它们
    people.push_back(info); // 将此记录追加到people末尾
}
```

- 使用`ostringstream`

  筛选上面的无效号码。每个人，验证完所有电话号码才进行输出。

```cpp
for (const auto &entry : people) { // 多people中每一项
    ostringstream formatted, badNums; // 每个循环步创建的对象
    for (const auto &nums : entry.phones) { // 对每个数
    	if (!valid(nums)) {
      	  badNums << " " << nums; // 将数的字符串形式存入badnums
    	} else
        // 将格式化的字符串"写入" formatted
        formatted << " " << format(nums);
    }
		}
    if (badNums.str().empty()) // 没有错误的数
        os << entry.name << " " // 打印名字
        << formatted.str() << endl; // 和格式化的数
    else // 否则，打印名字和错误的数
        cerr << "input error: " << entry
    << " invalid number(s) " << badNums.str() << endl;
}
```

