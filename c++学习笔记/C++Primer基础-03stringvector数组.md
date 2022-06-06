# C++Primer基础: string、vector、数组

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
string s3(4, 'K');  // s3 = "KKKK"
string s4("12345", 1, 3);  //s4 = "234"，即 "12345" 的从下标 1 开始，长度为 3 的子串
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
    
*   **string对象还可以用`append`进行拼接**

    ```cpp
    string s1("123"), s2("abc");
    s1.append(s2);  // s1 = "123abc"
    s1.append(s2, 1, 2);  // s1 = "123abcbc"
    s1.append(3, 'K');  // s1 = "123abcbcKKK"
    s1.append("ABCDE", 2, 3);  // s1 = "123abcbcKKKCDE"，添加 "ABCDE" 的子串(2, 3)
    ```
    
*   **string对象的比较**

    可以用`<、<=、==、!=、>=、>`运算符比较string对象。除此之外，string对象还可以用`compare`函数比较。
    
    - 小于0表示当前的字符串小;
    
    - 等于0表示两个字符串相等;
    
    - 大于0表示当前字符串大;
    
      ```cpp
      string s1("hello"), s2("hello, world");
      int n = s1.compare(s2);
      n = s1.compare(1, 2, s2, 0, 3);  //比较s1的子串 (1,2) 和s2的子串 (0,3)
      n = s1.compare(0, 2, s2);  // 比较s1的子串 (0,2) 和 s2
      n = s1.compare("Hello");
      ```
    
*   **遍历字符串中每个字符**

    ```cpp
    string str("Some string");
    for(auto c:str):
        cout<<c<<endl;
    ```
    
*   **使用for语句改变字符串中的字符**

    ```cpp
    string s("Hello world!!!");
    for(auto &c:s)
        c=touuper(c);
    count << s << endl;
    ```

![](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211127230618752.png)

- **求string对象的子串:`substr`,n表示起始位置,m表示子串的长度**

  ```cpp
  string substr(int n = 0, int m = string::npos) const;
  ```

  调用时，如果省略 m 或 m 超过了字符串的长度，则求出来的子串就是从下标 n 开始一直到字符串结束的部分。例如：

  ```cpp
  string s1 = "this is ok";
  string s2 = s1.substr(2, 4);  // s2 = "is i"
  s2 = s1.substr(2);  // s2 = "is is ok"
  ```

- **交换两个string对象的内容**

  ```cpp
  string s1("West”), s2("East");
  s1.swap(s2);  // s1 = "East"，s2 = "West"
  ```

- **查找子串和字符**

  string 类有一些查找子串和字符的成员函数，它们的返回值都是子串或字符在 string 对象字符串中的位置（即下标）。如果查不到，则返回`string::npos`。`string: :npos`是在 string 类中定义的一个静态常量。这些函数如下：
  - `find`: 从前往后查找子串或字符出现的位置。

  - `rfind`: 从后往前查找子串或字符出现的位置。

  - `find_first_of`: 从前往后查找何处出现另一个字符串中包含的字符。例如：`s1.find_first_of("abc");`<mark style="color:blue">**//查找s1中第一次出现"abc"中任一字符的位置**</mark>

  - `find_last_of`:从后往前查找何处出现另一个字符串中包含的字符。

  - `find_first_not_of`:从前往后查找何处出现另一个字符串中没有包含的字符;

  - `find_last_not_of`:从后往前查找何处出现另一个字符串中没有包含的字符;

    ```cpp
    string s1("Source Code");
    int n;
    if ((n = s1.find('u')) != string::npos) //查找 u 出现的位置
        cout << "1) " << n << "," << s1.substr(n) << endl; //输出 l)2,urce Code
    if ((n = s1.find("Source", 3)) == string::npos) //从下标3开始查找"Source"，找不到
        cout << "2) " << "Not Found" << endl;  //输出 2) Not Found
    if ((n = s1.find("Co")) != string::npos) //查找子串"Co"。能找到，返回"Co"的位置
        cout << "3) " << n << ", " << s1.substr(n) << endl; //输出 3) 7, Code
    if ((n = s1.find_first_of("ceo")) != string::npos) //查找第一次出现或 'c'、'e'或'o'的位置
        cout << "4) " << n << ", " << s1.substr(n) << endl; //输出 4) l, ource Code
    if ((n = s1.find_last_of('e')) != string::npos) //查找最后一个 'e' 的位置
        cout << "5) " << n << ", " << s1.substr(n) << endl;  //输出 5) 10, e
    if ((n = s1.find_first_not_of("eou", 1)) != string::npos)//从下标1开始查找第一次出现非 'e'、'o' 或 'u' 字符的位置
        cout << "6) " << n << ", " << s1.substr(n) << endl; //输出 6) 3, rce Code
    
    #最简单的去除首尾空格的方法
    std::string str01("  no empty space   ");
    str01.erase(0, str01.find_first_not_of(' '));
    str01.erase(str01.find_last_not_of(' ') + 1); //不是很严谨
    
    return 0;
    ```

- **替换子串:replace,一般需要和find结合使用**

  ```cpp
  string s1("Real Steel");
  s1.replace(1, 3, "123456", 2, 4);  //用 "123456" 的子串(2,4) 替换 s1 的子串(1,3)
  cout << s1 << endl;  //输出 R3456 Steel
  
  string s2("Harry Potter");
  s2.replace(2, 3, 5, '0');  //用 5 个 '0' 替换子串(2,3)
  cout << s2 << endl;  //输出 Ha0000 Potter
  
  int n = s2.find("00000");  //查找子串 "00000" 的位置，n=2
  s2.replace(n, strlen("00000"), "XXX");  //将子串(n,5)替换为"XXX"
  cout << s2 < < endl;  //输出 HaXXX Potter
  ```

- **删除子串:erease,一般需要和find结合使用**

  ```cpp
  string s1("Real Steel");
  s1.erase(1, 3);  //删除子串,位置1开始以及后面的三个字符，此后 s1 = "R Steel"
  s1.erase(5);  //删除下标5及其后面的所有字符，此后 s1 = "R Ste"
  
  #最简单的去除左边空格的方法
  std::string str01("  no empty space");
  str01.erase(0, str01.find_first_not_of(' ')-0);
  
  #和std::remove_if结合使用
  std::string str("example.!,");
  str.erase(std::remove_if(str.begin(), str.end(),[](unsigned char c) { return std::ispunct(c); }),str.end());
  //最后str中保存example
  ```

- **插入字符串:insert**

  ```cpp
  string s1("Limitless"), s2("00");
  s1.insert(2, "123");  //在下标 2 处插入字符串"123"，s1 = "Li123mitless"
  s1.insert(3, s2);  //在下标 2 处插入 s2 , s1 = "Li10023mitless"
  s1.insert(3, 5, 'X');  //在下标 3 处插入 5 个 'X'，s1 = "Li1XXXXX0023mitless"
  
  用erease + insert 实现replace。将hello 替换成 hi
  std::string str01 = "你好,hello luke";
  auto start = str01.find("hello");
  str01.erase(start, strlen("hello"));
  str01.insert(start, "hi");
  std::cout << "str01==" << str01 << std::endl;
  ```

- **将string作为流处理**

  ```cpp
  string src("Avatar 123 5.2 Titanic K");
  istringstream istrStream(src); //建立src到istrStream的联系
  string s1, s2;
  int n;  double d;  char c;
  istrStream >> s1 >> n >> d >> s2 >> c; //把src的内容当做输入流进行读取
  
  ostringstream ostrStream;
  ostrStream << s1 << endl << s2 << endl << n << endl << d << endl << c <<endl;
  cout << ostrStream.str();
  输出:
  Avatar
  Titanic
  123
  5.2
  K
    
  #按照空格切割字符串
  std::string _str01("zhangsna lisi wangwu tinghao");
  std::stringstream _ss(_str01);
  std::vector<std::string> _v01;
  std::string word;
  while (_ss >> word) {
    _v01.push_back(word);
  }
  
  #按照某个字符切割字符串
  std::vector<std::string> stringSplit(const std::string &s, const char ch) {
    std::vector<std::string> ret;
    std::stringstream ss(s);
    std::string segment;
    while (std::getline(ss, segment, ch)) {
      ret.push_back(segment);
    }
    return ret;
  }
  ```

- 用`STL`算法操作string对象

  ```cpp
  std::string s("afgcbed");
  std::string::iterator p = std::find(s.begin(), s.end(), 'c');
  if (p!= s.end())
      std::cout << p - s.begin() << std::endl;  //输出 3
  std::sort(s.begin(), s.end());
  std::cout << s << std::endl;  //输出 abcdefg
  
  #更合适的去除左边空格的方法
  static void ltrimSpace(std::string &s) {
    s.erase(s.begin(), std::find_if_not(s.begin(), s.end(), [](unsigned char ch) {
              return std::isspace(ch);
            }));
  }
  ```

  

### vector

* vector能容纳大多数类型的对象作为元素，但因为引用不是对象。因此**不存在包含 引用的vector**;

*   vector初始化

    ![image-20211127231245230](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211127231245230.png)
    
    ```cpp
    #错误 
    std::vector<std::string> _v01(5); #一开始就插入了5个元素
    _v01.push_back("a");
    _v01.push_back("b");
    结果: [,,,,,a,b]
    
    #正确,一开始就知道要插入多少个元素的话.可以避免多次内存分配
    std::vector<std::string> _v01(5);
    _v01[0] = "a";
    _v01[1] = "b";
    _v01[2] = "c";
    _v01[3] = "d";
    _v01[4] = "e";
    ```
    
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
    
*   **指定位置插入一个 or 多个元素`insert()`(性能不好，中间插入尽量用list)**

    - `iterator insert(pos,elem)` 在迭代器pos指定的位置之前插入一个新元素elem,并返回表示新插入元素位置的迭代器;
    
    - `iterator insert(pos,n,elem)` 在迭代器pos指定的位置之前插入n个新元素elem,并返回表示第一个新插入元素位置的迭代器;
    
    - `iterator insert(pos,first,last)` 在迭代器pos指定的位置之前,插入其他容器中位于`[first,last)`区域的所有元素,并返回表示第一个新插入元素位置的迭代器;
    
    - `iterator insert(pos,initlist)` 在迭代器pos指定的位置之前,插入初始化列表(用大括号`{}`括起来的多元素，中间逗号隔开),并返回表示第一个新插入元素位置的迭代器;
    
      ```cpp
      std::vector<int> demo{1,2};
      //第一种格式用法
      demo.insert(demo.begin() + 1, 3);//{1,3,2}
      
      //第二种格式用法
      demo.insert(demo.end(), 2, 5);//{1,3,2,5,5}
      
      //第三种格式用法
      std::array<int,3>test{ 7,8,9 };
      demo.insert(demo.end(), test.begin(), test.end());//{1,3,2,5,5,7,8,9}
      
      //第四种格式用法
      demo.insert(demo.end(), { 10,11 });//{1,3,2,5,5,7,8,9,10,11}
      ```
    
*   **删除元素的几种方式**

    * `pop_back()`:删除vector容器中最后一个元素，该容器的size会减1，但capacity不会改变。**该函数不会返回什么,如果最后的元素需要的话，最好利用`back()`提前取走**
    
      ```cpp
      vector<int>demo{ 1,2,3,4,5 };
      demo.pop_back(); // 1 2 3 4
      ```
    
    * `erease(pos)`:删除vector容器中pos迭代器指定位置处的元素，并返回指向被删除元素下一个位置元素的迭代器。该容器大小size-1，capacity不会发生改变;
    
      ```cpp
      vector<int>demo{ 1,2,3,4,5 };
      auto iter = demo.erase(demo.begin() + 1);//删除元素 2,此时demo中还剩 1 3 4 5
      cout << endl << *iter << endl; //此时iter指向3
      ```
    
    * `erease(beg,end)`: 删除vector容器中位于迭代器`[beg,end)`指定区域内的所有元素，并返回指向删除区域下一个位置元素的迭代器。该容器大小size会减少，capacity不会变;
    
      ```cpp
      std::vector<int> demo{ 1,2,3,4,5 };
      //删除 2、3
      auto iter = demo.erase(demo.begin()+1, demo.end() - 2); //此时demo中保存的数据 1 4 5
      ```
    
    * `std::remove(beg,end,ele)`:删除容器中所有和指定元素相等的元素，并返回指向最后一个元素下一个位置的迭代器。**调用该函数不会改变容器的大小和容量**
    
      ```cpp
      vector<int>demo{ 1,3,3,4,3,5 };
      auto iter = std::remove(demo.begin(), demo.end(), 3); //此时demo中数据为1 4 5 4 3 5
      //输出结果为 1 4 5,注意用iter
      for (auto first = demo.begin(); first < iter;++first) {
          cout << *first << " ";
      }
      ```
    
      **<mark style="color:red">注意，对容器执行完remve()函数之后，该函数并没有改变容器原来的大小和容量，因此无法使用之前的方法遍历容器，需要像下面程序中那样，借助remove()返回的迭代器完成正确的遍历。</mark>**
    
      > remove() 的实现原理是，在遍历容器中的元素时，一旦遇到目标元素，就做上标记，然后继续遍历，直到找到一个非目标元素，即用此元素将最先做标记的位置覆盖掉，同时将此非目标元素所在的位置也做上标记，等待找到新的非目标元素将其覆盖。因此，如果将上面程序中 demo 容器的元素全部输出，得到的结果为`1 4 5 4 3 5`。
    
      **既然通过 remove() 函数删除掉 demo 容器中的多个指定元素，该容器的大小和容量都没有改变，其剩余位置还保留了之前存储的元素。我们可以使用 erase() 成员函数删掉这些 "无用" 的元素。**
    
      ```cpp
      vector<int>demo{ 1,3,3,4,3,5 };
      auto iter = std::remove(demo.begin(), demo.end(), 3); //此时demo中数据为1 4 5 4 3 5
      demo.erase(iter, demo.end()); //此时demo中数据为1 4 5
      ```
    
    * `std::remove_if()`
    
      ```cpp
      std::vector<int> _v01={1,2,3,4,5};
      _v01.erase(std::remove_if(_v01.begin(), _v01.end(),[](const int &a) { return a > 3; }),_v01.end()); //_v01中数据 1 2 3
      ```
    
    * `clear()`:删除vector中所有元素，使其变成一个空的vector。该函数会改变vector的大小，但不会改变其容量。
    
*   **如何避免vector进行不必要的扩容?**

    ```cpp
    // 反例
    vector<int>myvector;
    for (int i = 1; i <= 1000; i++) //需要2^n > 1000  n次扩容
    {
        myvector.push_back(i);
    }
    
    //正例
    vector<int>myvector;
    myvector.reserve(1000);  //一次扩容
    cout << myvector.capacity();
    for (int i = 1; i <= 1000; i++)  //无需扩容
    {
        myvector.push_back(i);
    }
    ```
    
*   **不要使用`vector<bool>`，尽量使用`bitset`**

    原因: [谈vector<bool>的特殊性——为什么它不是STL容器](https://blog.csdn.net/haolexiao/article/details/56837445)

### deque

- deque擅长从序列的头部、尾部删除元素，时间复杂度都是`O(1)`。**不擅长在序列中添加 或 删除元素**
- **注意:deque容器中存储元素并不能保证所有元素都在连续的内存空间中。但还是可以用下标访问，如`que[0]=5、que.at(0)=5`;**

