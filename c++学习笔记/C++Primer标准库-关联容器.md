## C++Primer标准库: 关联容器

`map`和`set`：

- 允许重复关键字的容器的名字中都包含单词`multi`;
- 不保持关键字按顺序存储的容器的名字都以单词`unordered`开头;
- 因此`unordered_multi_set` 是一个允许重复关键字的，元素无序保存的集合;

`map`和`multimap`定义在头文件`map`中;
`set`和`multiset`定义在头文件`set`中;
无序容器则定义在头文件`unordered_map`和`unordered_set`中。

![2445987c54d19846085c75de56be6f35](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/E142D2A8-3C2B-4736-8B3B-E61E06972859.png)


**使用map**
```cpp
void mapTest(){
    map<string,size_t> word_count;
    string word;
    while(cin >> word)
        ++ word_count[word];
    
    for(const auto &w:word_count)
        cout << w.first << " occurs " << w.second << (( w.second > 1 ? " times ":"time")) << endl;
}
```
当从map中提取一个元素时，会得到一个`pair`类型的对象。简单来说，pair就是一个模板类型，保存两个名为`first`和`second`的数据成员。`first`保存关键字，`second`保存对应的值。

**使用set**

```cpp
// 统计输出中每个单词出现的此时，这些单词不包含 The But等
map<string, size_t> word_count; // empty map from string to size_t
set<string> exclude = {"The", "But", "And", "Or", "An", "A","the", "but", "and", "or", "an", "a"};
string word;
while (cin >> word)
    // count only words that are not in exclude
    if (exclude.find(word) == exclude.end())
        ++word_count[word]; // fetch and increment the counter for word
```

问题: 上面的程序，如果我们需要忽略大小写和标点，例如，"example."、"example,"和"Example"。应该怎么处理:
```cpp
str.erase(std::remove_if(str.begin(), str.end(),[](unsigned char c) { return std::ispunct(c); }),str.end());
remove_if的作用是将所有满足条件的元素移动到容器的最左边，str.erease用来做清理
```

**定义关联容器**

```cpp
map<string, size_t> word_count; // 空容器
// 列表初始化
set<string> exclude = {"the", "but", "and", "or", "an", "a",
                    "The", "But", "And", "Or", "An", "A"};
// 三个元素；authors将姓隐射为名
map<string, string> authors = { {"Joyce", "James"},
                        {"Austen", "Jane"},
                        {"Dickens", "Charles"} };

vector<int> ivec;
for (vector<int>::size_type i = 0; i != 10; ++i) {
    ivec.push_back(i);
    ivec.push_back(i); // 每个数字重复保存
}
// iset包含来自ivec的不重复的元素；miset包含所有20个元素
set<int> iset(ivec.cbegin(), ivec.cend());
multiset<int> miset(ivec.cbegin(), ivec.cend());
```

#### pair类型
在头文件`utility`中。
```cpp
pair<string, string> anon; // holds two strings
pair<string, size_t> word_count; // holds a string and an size_t
pair<string, vector<int>> line; // holds string and vector<int>

pair<string, string> author{"James", "Joyce"};
```
与其他标准库类型不同，`pair`的数据成员是`public`的。两个成员分别命名为`first`和`second`。
```cpp
// print the results
cout << w.first << " occurs " << w.second
    << ((w.second > 1) ? " times" : " time") << endl;
```
![dbd2ae3432296f820bdb30c53ffc2be4](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/0D1AD4C7-87D7-4EF4-954A-CF98FCC291F8.png)
函数:

```cpp
pair<string, int>
process(vector<string> &v)
{
    // process v
    if (!v.empty())
        return {v.back(), v.back().size()}; // 列表初始化
    else
        return pair<string, int>(); // 隐士构造函数
}

还可以是:
if (!v.empty())
    return pair<string, int>(v.back(), v.back().size());
    
或者:
if (!v.empty())
    return make_pair(v.back(), v.back().size());
```

#### 关联容器操作
![2b2f3e1a468efc3b19fed7d00122a62e](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/F43DEDC6-4E21-430C-A063-5ACDA337613D.png)
![7db917383f0b0d2826d5d41074871ecf](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/9C91E63B-F37E-4461-9590-6566DE652745.png)
![836b203fd2964ea9c1b0be28b27c75bf](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/404683A0-83D0-49E0-9F39-E82591DCD9D5.png)

![image-20211130001630650](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211130001630650.png)

![139cc72a65a4e48414edecc1197dbb95](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/1B2D64E4-1840-4E78-B6B0-51C0956578EE.png)
对于`set`类型，`key_type`和`value_type`是一样的。set中保存的值就是关键字。
在一个map中，元素是一个键值对，即，每个元素是一个pair对象，包含一个关键字和关联的值。
由于我们不能改变一个元素的关键字，因此这些pair的关键字部分是`const`的

```cpp
set<string>::value_type v1; // v1 is a string
set<string>::key_type v2; // v2 is a string
map<string, int>::value_type v3; // v3 is a pair<const string, int>
map<string, int>::key_type v4; // v4 is a string
map<string, int>::mapped_type v5; // v5 is an int
```

#### 关联容器迭代器
**当我们解引用一个关联容器的迭代器时，我们会得到一个类型为容器的`value_type`的值的引用**。
对于map而言，`value_type`是一个`pair`类型，其`first`成员保存`const`的关键字，`second`成员保存值。

```cpp
// 获得指向word_count中一个元素的迭代器
auto map_it = word_count.begin();

// *map_it 是指向一个pair<const string,size_t>对象的引用
cout << map_it->first; // 打印此元素的关键字
cout << " " << map_it->second; // 打印此元素的值
map_it->first = "new key"; // error: 关键字是const的
++map_it->second; // ok: 我们可以通过迭代器改变元素
```
一定注意: map的`value_type`是一个pair，我们可以改变pair的值，但不能改变关键字成员的值。

**set的迭代器是const的**

```cpp
set<int> iset = {0,1,2,3,4,5,6,7,8,9};
set<int>::iterator set_it = iset.begin();
if (set_it != iset.end()) {
    *set_it = 42; // error: set中关键字是只读的
    cout << *set_it << endl; // ok: 可以读取关键字
}
```
那如何修改set中的元素呢?

可以先删除`erease()`、而后在插入`insert`。

**遍历关联容器**

```cpp
// 获取指向首元素的迭代器
auto map_it = word_count.cbegin();
// 比较当前迭代器和尾后迭代器
while (map_it != word_count.cend()) {
    // 解引用迭代器，打印键值对
    cout << map_it->first << " occurs "
    << map_it->second << " times" << endl;
    ++map_it; // 移动到下一个元素
}
```

**添加元素，向set中添加元素**
```cpp
vector<int> ivec = {2,4,6,8,2,4,6,8}; // ivec 有8个元素
set<int> set2; // empty 集合
set2.insert(ivec.cbegin(), ivec.cend()); // set2 有4个元素
set2.insert({1,3,5,7,1,3,5,7}); // set2 现在有8个元素
```

**添加元素，向map中添加元素**
想map中进行insert操作时，必须是`pair`类型。
```cpp
word_count.insert({word, 1});
word_count.insert(make_pair(word, 1));
word_count.insert(pair<string, size_t>(word, 1));
word_count.insert(map<string, size_t>::value_type(word, 1));

#使用emplace()插入时,从返回值很方便判断插入成功还是失败
pair<map<string,size_t>::iterator, bool> ret =word_count.emplace("hello",4);
cout << "1、ret.iter = <{" << ret.first->first << ", " << ret.first->second << "}, " << ret.second << ">" << endl;
```
![3a885bdf012d16d04b73466ef8722336](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/E79A2F32-4171-4E2E-AA93-C7005295D766.png)

**检测insert的返回值**
`insert`返回的值依赖容器类型和参数。对于不包含重复关键字的容器，<mark style="color:red;">**添加单一元素的insert和emplace版本返回一个pair，告诉我们插入操作是否成功。pair的first成员是一个迭代器，指向具有给定关键字的元素；second成员是一个bool值，指出元素插入成功还是已经存在于容器中。**</mark>
如果关键字已经在容器中，则insert什么事情也不做，而且返回值中bool部分为`false`；否则Bool值为`true`。

```c++
// more verbose way to count number of times each word occurs in the input
map<string, size_t> word_count; // empty map from string to size_t
string word;
while (cin >> word) {
    // 插入一个元素，关键字等于word，值为1
    // 若word已在word_count中，insert什么也不做
    auto ret = word_count.insert({word, 1});
    if (!ret.second) // word 已在 word_count 中
        ++ret.first->second; // 递增计数器
}
```

**向multiset或multimap中添加元素**
现在：如果我们想建立作者到他所著书籍题目的映射，在这种情况下，每个作者可能有多个条目，因此我们应该使用`multimap`而不是`map`。

```cpp
multimap<string, string> authors;
// 插入第一个元素，key是 Barth, John
authors.insert({"Barth, John", "Sot-Weed Factor"});
// ok: 添加第二个元素，关键字是 Barth, John
authors.insert({"Barth, John", "Lost in the Funhouse"});
// or
authors.insert(pair<string,string>{"Barth, John", "Lost in the Funhouse"});
// or
authors.emplace("Barth, John", "Lost in the Funhouse");
```
对允许重复关键字的容器中插入元素，insert操作返回一个指向新元素的迭代器。这里不会返回bool值，因为insert总是成功的。

#### 删除元素
和关联容器类似，我们可以通过`erase`，参数是一个迭代器，删除一个元素 或者一个元素范围。

注意：这里的删除会将所有匹配给定关键字的元素删除，返回实际删除的元素的数量。
```cpp
// erase 一个关键字，返回删除的元素数量
if (word_count.erase(removal_word))
    cout << "ok: " << removal_word << " removed\n";
else cout << "oops: " << removal_word << " not found!\n";
```
![76305861e45f7c8c57542b4f6663ad64](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/99BE5D88-C7B7-457B-996C-197ECE31AB49.png)

#### map的下标操作
注意：<mark style="color:red;">**总体来说，对map尽量不要用下标操作比较好。使用也仅在插入新元素时使用。**</mark>
- 同时，我们不能对一个`multimap`或一个`unordered_multimap`进行下标操作，因为这些容器中可能有多个值与关键字相关联。
- <mark style="color:red;">**map如果进行下标操作，如果关键字不在map中，则会为他创建一个新元素并插入到map中。关联值将进行值初始化**</mark>。(这是一个奇葩的设计)
```cpp
map <string, size_t> word_count; // empty map
// 插入一个关键字为Anna的元素，关联值进行值初始化，并将1赋予它
word_count["Anna"] = 1;
```
![48645dcfdf4bb0c86cdacbef20826bae](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/BD1E7C17-7776-459B-A1A1-DD44E8AB2FAB.png)

**使用下标操作的返回值**
当对一个map进行下标操作时，会获得一个`mapped_type`对象；
当解引用一个map迭代器时，会得到一个`value_type`对象。

```cpp
cout << word_count["Anna"]; // 使用Anna作为下标提取元素，会打印出1
++word_count["Anna"]; // 提取元素，将其增1
cout << word_count["Anna"]; // 提取元素并打印它；会打印出2
```

#### 访问元素
```cpp
set<int> iset = {0,1,2,3,4,5,6,7,8,9};
iset.find(1); // 返回一个迭代器，指向 key ==1 的元素
iset.find(11); // 返回一个迭代器，其值等于iset.end()
iset.count(1); // returns 1
iset.count(11); // returns 0
```
![de28598d73698b1fd5944ad3ea87c2ea](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/11B6E178-8729-4FBC-9BC6-0B24A915E5CF.png)

`lower_bound()`、`upper_bound()`一般用在`multimap`、`multiset`中。一个是大于等于、一个是大于，刚好组成一个 左闭右开 区间`[)`

- 对map使用find代替下标操作；

**在multimap或multiset中查找元素**
如果一个`multimap`或`multiset`中有多个元素具有给定关键字，则这些元素在容器中会**相邻存储**。

问题：给一个从作者到著作题目的映射，我们可能想打印一个特定作者的所有拙作。下面有是那种方案来解决该问题:
- 方案一:`find` + `count`
```c++
string search_item("Alain de Botton"); // 要查找的作者
auto entries = authors.count(search_item); // 元素的数量
auto iter = authors.find(search_item); // 此作者的第一本书
// 用一个循环查找此作者的所有著作
while(entries) {
    cout << iter->second << endl; // 打印每个题目
    ++iter; // 前进到下一本书
    --entries; // 记录已经打印了多少本书
}
```
- 方案二: 迭代器解决思路
a. 如果关键字在容器中，`lower_bound`返回的迭代器将指向第一个具有给定关键字的元素；
b. `upper_bound`返回的迭代器则指向最后一个匹配给定关键字元素之后的位置。
c. 如果元素不在`multimap`中，则`lower_bound`和`upper_bound`会返回相等的迭代器——指向一个不影响排序的关键字插入位置。
d. 此时`lower_bound`和`upper_bound`会得到一个迭代器范围。
```c++
// authors 和 search_item的定义，与前面的程序一样
// beg and end 表示对于此作者的元素的范围
for (auto beg = authors.lower_bound(search_item),
        end = authors.upper_bound(search_item);
        beg != end; ++beg)
cout << beg->second << endl; // print each title
```
- 方案三: `equal_range`函数
`equal_range`接受一个关键字，返回一个迭代器`pair`。
如果关键字存在，则第一个迭代器指向第一个与关键字匹配的的元素；第二个迭代器指向 最后一个匹配元素之后的位置；
如果未找到匹配元素，则两个迭代器都指向关键字可以插入的位置。
```c++
// authors 和 search_item 的定义，与前面的程序一样
// pos 保存迭代器对，表示与关键字匹配的元素范围
for (auto pos = authors.equal_range(search_item);
        pos.first != pos.second; ++pos.first)
    cout << pos.first->second << endl; // print each title
```

<mark style="color:red">**multimap还可以定义第三个参数:key比较方法**</mark>

```cpp
std::multimap<std::string, int, std::less<std::string>> _mm01;
// std::multimap<std::string, int, std::greater<std::string>> _mm01;
_mm01.emplace("bb", 22);
_mm01.emplace("aa", 11);
_mm01.emplace("dd", 44);
_mm01.emplace("cc", 33);
for (auto &item : _mm01)
  std::cout << "[" << item.first << "," << item.second << "]" << std::endl;

#自定义key比较方法
struct myCmp {
  bool operator()(const std::string &a, const std::string &b) const {
    return a < b;
  }
};
std::multimap<std::string, int, myCmp> _mm01;
_mm01.emplace("bb", 22);
_mm01.emplace("aa", 11);
_mm01.emplace("dd", 44);
_mm01.emplace("cc", 33);
```

