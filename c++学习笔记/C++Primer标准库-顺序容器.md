## C++Primer 标准库: 顺序容器

顺序容器在以下几个方面都有不同的性能折中:

- 向容器中添加或从容器中删除元素的代价；

- 非顺序访问容器中元素的代价;

![image-20211129164548276](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129164548276.png)

**确定使用哪种顺序容器**

- 除非很好选择其他容器的理由，否则应使用`vector`;

- 如果程序要求随机访问元素，则应使用`vector`、`deque`;

- 如果程序要在容器中间插入 或 删除元素，则使用`list`、`forward_list`;

- 如果程序需要在头部位置插入或者删除元素，但不会再中间位置进行插入或删除操作，则使用`deque`;
  - 首先，确定是否真的需要在容器中间位置插入元素。当处理输入数据时，通常可以很容易地向`vector`追加数据，然后再调用标准库的sort函数重排容器中元素，从而避免在中间位置添加元素；
  - 如果必须在中间位置插入元素，考虑在输入阶段使用`list`，一旦输入完成，将`list`中的内容拷贝到`vector`中;

如果程序既需要随机访问元素，又需要在容器中间位置插入元素？咋办？

那取决于`list`、`forward_list`中访问元素与`vector`或`deque`中插入/删除元素的相对性能。

一般来说，应用中占有主导地位(执行的访问操作更多时候 or 插入/删除更多)决定了容器类型的选择。

#### 容器库概览

容器类型上操作形成了一种层次:

- 某些操作是所有容器类型都提供的；

- 另外一些操作仅针对顺序容器；

- 还有一些操作只适用于少部分容器类型； 

本节介绍的，是介绍对所有容器都适用的操作。

![image-20211129165143502](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129165143502.png)

![image-20211129165251864](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129165251864.png)

![image-20211129165317214](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129165317214.png)

![image-20211129165335346](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129165335346.png)

#### 迭代器

注意:`forward_list`迭代器不支持递减运算符(`--`);

- **迭代器的范围**

  左闭合区间(left-inclusive interval),其标准数学描述为:`[begin,end)`

  表示范围自`begin`开始，于`end`之前结束。

- **使用左闭合范围蕴含的编程假定:**

  - 如果`begin`与`end`相等，则范围为空;
  - 如果`begin`与`end`不等，则范围至少包含一个元素，且`begin`指向范围中的第一个元素;
  - 我们可以对`begin`递增若干次，使得`begin==end`;

  问题: 下面程序有何错误？应该如何处理？

  ```cpp
  list<int> lst1;
  list<int>::iterator iter1 = lst1.begin(),iter2 = lst1.end();
  while (iter1 < iter2) /* ... */
  ```

  答案: `iter1 < iter2`是错误的，迭代器类型是不能比较大小的，应该用`iter1 != iter2`。

  问题: 编写一个函数，接收一对指向`vector<int>`的迭代器和一个int值，在两个迭代器之间查找给定的值，返回一个bool值来确定是否找到？

  ```cpp
  bool findItemInIterRange(vector<int>::const_iterator begin,vector<int>::const_iterator end,int target){
      while(begin != end){
          if(*begin == target){
              return true;
          }
          begin++;
      }
      return false;
  }
  ```

#### 容器类型成员

每个容器都定义了很多类型。上面我们已经使用过其中三种: `size_type`、`iterator`、`const_iterator`。

**反向迭代器**:所有操作的含义都是颠倒的，如反向迭代器中执行`++`操作，会得到上一个元素。后续介绍。

**类型别名**: 通过类型别名，我们可以在不了解容器中元素类型的情况下使用它。如果需要元素类型，则可以使用容器的`value_type`、如果需要元素中的一个引用，则我们可以使用`refrence`或`const_refrence`。

问题: 为了索引int的vector元素，应该使用什么类型？

答:`vector<int>::size_type`

问题: 为了读取string的list中的元素，应该使用什么类型？如果写入list，应该使用什么类型？

答: 读取:`list<string>::iterator || list<string>::const_iterator`;

写入:`list<string>::iterator`

#### 容器定义和初始化

![image-20211129170357199](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129170357199.png)

**将一个容器初始化为另一个容器的拷贝**

为了创建一个容器为另一个容器的拷贝，**两个容器的类型以及元素类型都必须匹配**。

不过，当传递迭代器参数来拷贝一个范围时，就不要求容器类型相同了。

```cpp
// 每个容器都有三个元素，用给定的初始化器进行初始化
list<string> authors = {"Milton", "Shakespeare", "Austen"};
vector<const char*> articles = {"a", "an", "the"};

list<string> list2(authors); // 正确: 类型匹配
deque<string> authList(authors); // error: 容器类型不匹配
vector<string> words(articles); // error: 元素类型必须匹配

// ok: 可以将const char*转换为string
forward_list<string> words(articles.begin(), articles.end());
```

注意哦: **如果是利用 两个迭代器 参数来初始化，两个迭代器 分别表示想要拷贝的第一个元素和尾元素之后的位置**。

```cpp
// 拷贝元素，直到(但不包括)it指向的元素
deque<string> authList(authors.begin(),it);
```

**与顺序容器大小相关的构造函数(不支持array)**

```cpp
vector<int> ivec(10, -1); // 10个int元素，每个都初始化为-1
list<string> svec(10, "hi!"); // 10个string元素，每个都初始化为"hi!"
```

注意: 只有顺序容器的构造函数才接受大小参数，关联容器并不支持。

**标准库aaray具有固定大小**

```cpp
array<int, 10> ia1; // 10个默认初始化的int
array<int, 10> ia2 = {0,1,2,3,4,5,6,7,8,9}; // 列表初始化
array<int, 10> ia3 = {42}; // ia3[0] is 42, 剩下的元素为0

int digs[10] = {0,1,2,3,4,5,6,7,8,9};
int cpy[10] = digs; // error: 内置数组不支持拷贝或赋值
array<int, 10> digits = {0,1,2,3,4,5,6,7,8,9};
array<int, 10> copy = digits; // ok: 只要数组类型匹配即合法
```

#### 赋值与swap

由于 右边对象的大小 可能与 左边运算对象的大小 不同。因此array类型不支持assign，也不允许用花括号包围的值列表进行赋值。

![image-20211129170736674](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129170736674.png)

**使用assign(仅顺序容器)**

```cpp
list<string> names;
vector<const char*> oldstyle;
names = oldstyle; // error: 容器类型不匹配
// ok: 可以讲const char*转换为string
names.assign(oldstyle.cbegin(), oldstyle.cend());

// 等价于 slist1.clear()
// 后跟 slist1.insert(slist1.begin(),10,"HiYa!")
list<string> slist1(1); // 一个元素，空string
slist1.assign(10, "Hiya!"); // 10个元素，每个都是 "Hiya!"
```

**使用swap**

swap操作交换两个相同类型容器的内容。

```cpp
vector<string> svec1(10); // vector with ten elements
vector<string> svec2(24); // vector with 24 elements
swap(svec1, svec2);
```

swap完成后，svec1将包含24个string元素，svec2将包含10个sttring。

除了array外，交换两个容器内容的操作是很快的行为——元素本身并未交换，swap只是交换了两个容器的内部结构。

swap不做任何元素的拷贝、删除、插入操作，因此可以保证在常数时间内完成。

元素不会被移动的事实意味着，除string外，指向容器的迭代器、引用、指针在swap后都不会失效。只是swap后，这些元素已经属于不同容器了。

如：假定`iter`在swap之前之指向`svec1[3]`的string，那么再swap之后它指向`svec2[3]`的元素。

注意：

- 对一个`string`调用swap会导致迭代器、引用和指针失效；

- swap两个`array`会真正交换它们的元素。

- 容器既提供了成员函数的`swap`,也提供了非成员版本的`swap`。

  **非成员版本的swap在泛型编程中非常重要，统一使用非成员版本的`swap`是一个好习惯。也就是尽量使用`swap(a,b)`、不要使用**`a.swap(b)`;

#### 容器大小操作

- `size()`返回容器中元素数目;

- `empty()`当size为0时返回true,否则false;

- `max_size()`返回一个大于等于该类型所能容纳的最大元素数的值；

- `forward_list`支持`max_size`和`empty`,但不支持`size()`;

### 顺序容器操作

上面一节介绍的是 顺序容器、关联容器都具有的操作，这一节主要介绍 顺序容器所特有的操作。

#### 向顺序容器中(非array)添加元素

![image-20211129171241741](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129171241741.png)

注意：容器元素是拷贝

当我们用一个对象来初始化容器时，或将一个对象插入到容器中时，实际上放入到容器中的对象值是一个拷贝，而不是对象本身。

就像我们给一个函数传递非引用参数一样。

**在容器中特定位置添加元素**

`push_back`、`push_front`就不说了。下面来说说`insert`:

```cpp
vector<string> svec;
list<string> slist;
// 等价于调用 slist.push_front("Hello");
slist.insert(slist.begin(), "Hello!");
// vector不支持push_front,但我们可以插入到begin()之前
// 警告：插入vector末尾以外的任何位置都很慢。
svec.insert(svec.begin(), "Hello!");
```

将元素插入到`vector`、`deque`、`string`中的任何位置都是合法的，然而这种做法很耗时。

**插入范围内元素**

```cpp
vector<string> v = {"quasi", "simba", "frollo", "scar"};
// 将v的随后两个元素添加到slist的开始位置
slist.insert(slist.begin(), v.end() - 2, v.end());
slist.insert(slist.end(), {"these", "words", "will",
"go", "at", "the", "end"});
// 运行是错误：迭代器表示要拷贝的范围，不能指向与目的位置相同的容器
slist.insert(slist.begin(), slist.begin(), slist.end());
```

**使用insert的返回值**

通过使用insert的返回值，我们可以在容器中特定位置反复插入一个元素:

```cpp
list<string> 1st;
auto iter = 1st.begin();
while (cin >> word)
    iter = 1st.insert(iter, word); // same as calling push_front
```

**使用emplace操作**

新标准引入了三个新成员——`emplace_front`、`emplate`和`emplace_back`，这些操作构造而不是拷贝元素。

当我们调用push或insert成员函数时，我们将元素类型的对象传递给他们，这些对象被拷贝到容器中。

而当我们调用一个emplace成员函数时，则是将参数传递给元素类型的构造函数。如:

```cpp
// 在c的末尾构造一个Sales_data对象
//使用三参数的Sales_data 构造函数
c.emplace_back("978-0590353403", 25, 15.99);
c.emplace_back(); // 使用Sales_data的默认构造函数
c.emplace(iter,"999-9999999999"); // 使用Sales_data(string)构造函数
c.emplace_front("999-999999999",25,15,99); // 使用Sales_data的接收一个ISBN、一个cout和一个price的构造函数。

// 错误: 没有接收三参数的push_back版本
c.push_back("978-0590353403", 25, 15.99);
// 正确: 创建一个临时的Sales_data对象传递给push_back
c.push_back(Sales_data("978-0590353403", 25, 15.99));
```

#### 访问元素

- 包括`array`在内的每个顺序容器都有一个`front`成员函数。如果容器中没有元素，访问操作的结果是未定义的；

- 包括`array`在内的每个顺序容器都有一个`front`成员函数，而除了`forward_list`之外的所有顺序容器都有一个`back`成员函数。

- `front`、`back`分别返回容器"首元素"、“尾元素”的引用;

```cpp
// 在调用front、back以及解引用迭代器以前，先检查容器是否为空
if (!c.empty()) {
    // val and val2 是c的第一个元素值的拷贝
    auto val = *c.begin(), val2 = c.front();
    
    // val3 and val4 是C中最后一个元素值的拷贝
    auto last = c.end();
    auto val3 = *(--last); // 不能递减forward_list迭代器
    auto val4 = c.back(); // forward_list 不支持
}
```

注意:

- 判断容器是否为空很重要；

- end返回的迭代器是不存在的，所以为了尾元素，必须首先递减次迭代器。

![image-20211129172410160](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129172410160.png)

**访问成员函数返回的都是引用**

![image-20211129172504361](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129172504361.png)

#### 删除元素

![image-20211129172555066](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129172555066.png)

删除元素的成员函数并不检查其参数。在删除元素之前，用户必须确保它是存在的。

**`pop_front`和`pop_back`成员函数**

- `pop_front`和`pop_back`成员函数分别删除首尾元素;

- `vector`和`string`不支持`pop_front`;

- `forward_list`不支持`pop_back`;

- 这些操作返回的都是`void`,如果你需要弹出这些元素的值，就必须在执行弹出操作前将其保存:

  ```cpp
  while (!ilist.empty()) {
      process(ilist.front()); // 对ilist的首元素进行一些处理
      ilist.pop_front(); // 完成处理后删除首元素
  }
  ```

**从容器内部删除一个元素**

注意:`erase`**返回的是指向删除的(最后一个)元素之后位置的迭代器**。

下面示例，删除list中的奇数元素:

```cpp
list<int> lst = {0,1,2,3,4,5,6,7,8,9};
auto it = lst.begin();
while (it != lst.end())
    if (*it % 2) // if the element is odd
        it = lst.erase(it); // erase this element,it不需要++
    else
        ++it;
```

注意: **`while`的条件必需写成`it != lst.end()`**

**删除多个元素**

```cpp
// 删除两个迭代器表示的范围内的元素
// 返回指向最后一个被删除元素之后位置的迭代器
// 注意: elem2是不会被删除的，它指向了我们删除的最后一个元素之后的位置
elem1 = slist.erease(elem1,elem2);
```

删除容器中所有元素:

```cpp
slist.clear();
slist.erease(slist.begin(),slist.end());
```

#### 改变容器大小(暂时不知道什么场景用)

除了`array`。

如果当前大小大于所要求的大小，容器后部的元素会被删除；

如果当前大小小于新大小，会将新元素添加到容器后部。

```cpp
list<int> ilist(10, 42); // 10个int，每个的值都是42
ilist.resize(15); // 将5个值为0的元素添加到ilist的末尾
ilist.resize(25, -1); // 将10个值为-1的元素添加到ilist的末尾
ilist.resize(5); // 从ilist末尾删除20个元素
```

#### 容器操作可能使得迭代器、引用、指针等失效

我们来看看使用`insert`、`erease`时，我们如何更新迭代器？

```cpp
// 傻瓜循环，删除偶数元素，复制每个奇数元素
vector<int> vi = {0,1,2,3,4,5,6,7,8,9};
auto iter = vi.begin(); // 调用begin而不是cbegin，因为我们要改变vi
while (iter != vi.end()) {
    if (*iter % 2) {
        iter = vi.insert(iter, *iter); // 复制当前元素
        iter += 2; // 向前移动迭代器，注意是两步，跳过当前元素以及插入到它之前的元素
    } else
        iter = vi.erase(iter); // 删除偶数元素 
        // erease后不应向前移动迭代器，iter指向我们删除的元素之后的元素
}
```

**<mark style="color:red;">不保存end返回的迭代器</mark>**

当我们添加/删除vector或string的元素后，或在deque中首元素之外任何位置添加/删除元素之后，原来end返回的迭代器总是会失效。

因此，添加或者删除元素的循环程序必须反复调用end，而不能在循环之前保存end返回的任何迭代器，并一直当做容器末尾使用。

![image-20211129173223016](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211129173223016.png)

