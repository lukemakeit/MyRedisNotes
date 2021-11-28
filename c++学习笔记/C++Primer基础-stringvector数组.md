# C++Primer基础: string、vector、数组

### C++Primer基础:string、vector、数组

### string

```cpp
#include <string>
using std::string;
或
using namespace std;
```

* 初始化

```cpp
string s1; //默认初始化,s1是空串
string s2(s1); //s2是s1的副本
string s2=s1; // 等价于s2(s1);
string s3("value"); //s3是字面值"value"的副本,除了字面值最后的那个空字符串外
string s3="value"; // 等价于s3("value")
string s4(n,'c'); //s4初始化为连续n个字符c组成的字符串
```

* `cin >> s`,如果输入的是" Hello World! ",则读取字符串s中的内容是"Hello" 没有任何空格。
* `getline()` 都去一整行,注意不会将换行符读到变量中

```cpp
string line;
while(getline(cin,line))
    cout << line <<endl;
return 0;
```

*   `line.size()`返回的是`string::size_type`类型，我们只能大致猜到`string::size_type`是无符号整数

    ```cpp
    auto len=line.size() 这样子获取string长度
    ```

    特别注意:**表达式中混用signed integer和unsigned integer很危险。比如n是一个具有负数值的int，则表达式`s,size()<n` 判断结果几乎肯定是true。因为负值n会自动转换成一个比较大的unsinged integer**


*   字面值和string相加: **必须确保每个加法运算符(`+`)的两侧的运算对象至少有一个string**

    ```cpp
    string s1="hello", s2="wolrd";
    string s3=s1 + “， ”+s2+‘\n’; 正确
    string s4=s1 + ","; 正确
    string s5="hello" + "," 错误：加法的两个运算对象都不是string
    string s6=s1+"," + "world"; 正确
    string s7="hello" + "," + s2; 错误
    ```

    为什么`string s6=s1+"," + "world";` 正确，而`string s7="hello" + "," + s2;`错误呢？

    因为`string s6=s1+"," + "world";` 等价于:

    ```cpp
    string tmp=s1+","; 正确
    s6=tmp + "world";  正确
    ```

    而`string s7="hello" + "," + s2;`中`("hello" + ", ") + s2` 括号中内容非法。

    特别注意: 因为某些历史原因，也为了和C兼容，所以C++语言中字符串字面值并不是string类型的对象。切记，**字符串字面值与string是不同的类型。**
*   遍历字符串中每个字符:

    ```cpp
    string str("Some string");
    for(auto c:str):
        cout<<c<<endl;
    ```
*   使用for语句改变字符串中的字符:

    ```cpp
    string s("Hello world!!!");
    for(auto &c:s)
        c=touuper(c);
    count << s << endl;
    ```

![](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211127230618752.png)

### vector

* vector能容纳大多数类型的对象作为元素，但因为引用不是对象。因此**不存在包含 引用的vector**;
*   vector初始化

    ![image-20211127231245230](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211127231245230.png)
*   添加元素`push_back()`:

    ```cpp
    示例一:
    vector<int> v2; //空的vector对象
    for (int i=0;i!=100;++i)
        v2.push_back(i);

    示例二:
    string word;
    vector<string> text;
    while (cin >> word){
        text.push_back(word);
    }
    ```
* 如果循环中有含有向vector对象添加元素的语句，则不能使用范围for循环。也就是说 **范围for语句中不应该改变其所遍历序列的大小**
*   vector一些其他操作

    ![image-20211127231359883](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211127231359883.png)
*   vector遍历:

    ```cpp
    vector<int> v{1,2,3,4,5,6,7,8,9};
    for (auto &i : v) // for each element in v (note: i是一个引用)
        i *= i; // square the element value
    for (auto i : v) // for each element in v
        cout << i << " "; // print the element
    cout << endl;
    ```
*   不能用下标形式添加元素

    ```cpp
    vector<int> ivec; 空对象
    for (decltype(ivec.size()) ix=0;ix!=10; ++ix)
        ivec[ix] = ix; //严重错误: ivec不包含任何元素
    ```
