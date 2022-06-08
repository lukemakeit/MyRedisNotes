## Effective STL， 30条有效使用STL的经验

原文: [Effective STL， 30条有效使用STL的经验](https://mp.weixin.qq.com/s/TZU2-LkePp4YsUnZN_HvoA)

1. **慎重选择STL容器类型**
    a) 确保自己了解每个容器的使用场景，特定的场景选择合适的容器类型
    b) 连续内存，支持下标访问，可考虑选择vector
    c) 频繁的在中间做插入或者删除操作，可考虑选择list
    d) 两者都有，可考虑使用deque

2. **确保容器中的对象拷贝正确而高效**
    a) 大家应该都知道，容器中存放的都是对象的拷贝，想要拷贝正确那就实现**拷贝构造函数**和**拷贝赋值运算符**
    b) 想要更高效，**可以使容器包含指针而不是对象，也可考虑智能指针**

3. **调用empty而不是检查size()是否为0**
    a) empty对所有的标准容器都是常数时间操作，而对一些list实现，size耗费线性时间

4. **如果容器中包含了通过new操作创建的指针，切记在容器对象析构前将指针delete掉**
    a) 其实就是为了避免资源泄漏
    b) 可以考虑在容器中存储shared_ptr

5. **慎重选择删除元素的方法**
    **a) 要删除容器中有特定值的所有对象**
    i. 如果容器是vector、string或deque，则使用erase-remove习惯用法
    ii. 如果容器是list，则使用list::remove
    iii. 如果容器是一个标准关联容器，则使用它的erase成员函数

  ```cpp
  vector<int>demo{ 1,3,3,4,3,5 };
  auto iter = std::remove(demo.begin(), demo.end(), 3); //此时demo中数据为1 4 5 4 3 5
  demo.erase(iter, demo.end()); //此时demo中数据为1 4 5
  ```

  **b) 要删除容器中满足特定条件的所有对象**
  i. 如果容器是vector、string或deque，则使用erase-remove_if习惯用法

  ```cpp
  std::vector<int> _v01={1,2,3,4,5};
  _v01.erase(std::remove_if(_v01.begin(), _v01.end(),[](const int &a) { return a > 3; }),_v01.end()); //_v01中数据 1 2 3
  ```

  ii. 如果容器是list，则使用list::remove_if

  ```cpp
  std::list<int> _l01 = {1, 2, 3, 4, 5};
  _l01.remove_if([](const int &a) { return a > 3; });
  ```

  iii. 如果容器是一个标准关联容器，则使用remove_copy_if和swap，或者写一个循环来遍历容器中的元素，记住当把迭代器传给erase时，要对它进行后缀递增

  **c) 要在循环内部做某些操作**
  i. <mark style="color:red">**如果容器是一个标准序列容器，则写一个循环来遍历容器中的元素，记住每次调用erase时，要用它的返回值更新迭代器**</mark>

  ```cpp
  for (auto i = c.begin(); i != c.end();) {
      if (xxx) {
          i = c.erase(i);    
      }
      else ++i;
  }
  ```

  ii. <mark style="color:red">**如果容器是一个标准关联容器，则写一个循环来遍历容器中的元素，记住当把迭代器传给erase时，要对迭代器做后缀递增**</mark>

  ```cpp
  for (auto i = c.begin(); i != c.end();) {
      if (xxx) {
          c.erase(i++);    
      }
      else ++i;
  }
  ```

  **f)（!!!）现在可以统一使用返回值更新迭代器方式**

6. **vector等容器考虑使用reserve来避免不必要的重新分配**

  i. 若能明确知道或预计容器最终有多少元素，可使用reverse，预留适当大小的空间;
  ii. 先预留足够大的空间，然后，当把所有数据都加入以后，再去除多余的空间;

7. **避免使用vector存储bool**
    a)  有两点：
   i. 它不是一个STL容器，不能取元素的地址
   ii. 它不存储bool
    b) 可以用deque和bitset来替代

8. <mark style="color:red">**为包含指针的关联容器指定比较类型**</mark>

  a)  容器里面存储的都是指针，但是由于是关联容器，需要进行比较，但默认的比较（比较指针）一般不是我们想要的行为;
  b)  所以需要指定比较类型，自定义比较行为;

9. <mark style="color:red">**总是让比较函数在等值情况下返回false**</mark>

  参考文章: [P1级公司故障，年终奖不保](https://mp.weixin.qq.com/s/TZU2-LkePp4YsUnZN_HvoA)

10. <mark style="color:red">**切勿直接修改set 或 multiset中的键**</mark>

  a) **如果改变了键,那么可能破坏了该容器的顺序，再使用该容器可能导致不确定的结果。可以考虑先删除，修改后，再插入**;

  b) **为何标题是切勿修改set,而不是切勿修改map中的键呢?因为map中的键本来就是const K, 无法修改**

11. **当效率至关重要时，请在`map::operator[]`与`map::insert`之间谨慎选择**

   a) 当向map中添加元素时，优先选用insert而不是`operator[]`

   b) 当更新map中的值时，优先选用`operator[]`

12. **对于逐个字符的输入,考虑使用`istreambuf_iterator`**

   a)  `istreambuf_iterator()`性能一般优于`istream_iterator`

   b) `istreambuf_iterator`不会跳过任何字符

   ```cpp
   istream inputFile("xxx.txt");
   string str(istreambuf_iterator<char>(inputFile), istreambuf_iterator<char>());
   ```

13. **了解各种排序相关的选择**

   a) 重点关注以下几项

   - `sort`

   - `stable_sort`

   - `partial_sort`: 比如有100w个元素的容器，我们只想从中提取出值最小的10个元素，如何实现?

     语法:

     ```cpp
     //按照默认的升序排序规则，对 [first, last) 范围的数据进行筛选并排序
     void partial_sort (RandomAccessIterator first,RandomAccessIterator middle,RandomAccessIterator last);
     
     //按照 comp 排序规则，对 [first, last) 范围的数据进行筛选并排序
     void partial_sort (RandomAccessIterator first,RandomAccessIterator middle,RandomAccessIterator last,Compare comp);
     ```

     - `partial_sort()`函数会以交换元素存储位置的方式实现部分排序。具体来说，`partial_sort()`会将`[first,last)`范围内最小(或最大)的`middle-first`个元素移动到`[first,middle)`区域中，并对这部分元素做升序(或降序)排序;
     - `partial_sort()`要求容器支持随机方法。也就意味着，该函数只适用于`array`、`vector`、`deque`3类容器;
     - 当选用默认的升序排序规则时,容器中存储的元素类型必须支持`<`小于运算符;同样,如果选用标准库提供的其它排序规则，元素类型也必须支持该规则底层实现所用的比较运算符；
     - partial_sort() 函数在实现过程中，需要交换某些元素的存储位置。因此,如果容器中存储的是自定义的类对象,则该类的内部必须提供移动构造函数和移动赋值运算符
     - `partial_sort()`函数实现排序的平均时间复杂度为`N*log(M)`,其中N指的是`[first,last)`范围的长度，`M`值的是`[first,middle)`范围的长度。

     ```cpp
     //以函数对象的方式自定义排序规则
     class mycomp2 {
     public:
         bool operator() (int i, int j) {
             return (i > j);
         }
     };
     
     std::vector<int> myvector{ 3,2,5,4,1,6,9,7};
     
     //以默认的升序排序作为排序规则，将 myvector 中最小的 4 个元素移动到开头位置并排好序
     std::partial_sort(myvector.begin(), myvector.begin() + 4, myvector.end());
     cout << "第一次排序:\n";
     for (std::vector<int>::iterator it = myvector.begin(); it != myvector.end(); ++it)
         std::cout << *it << ' ';
     cout << "\n第二次排序:\n";
     
     // 以指定的 mycomp2 作为排序规则，将 myvector 中最大的 4 个元素移动到开头位置并排好序
     std::partial_sort(myvector.begin(), myvector.begin() + 4, myvector.end(), mycomp2());
     for (std::vector<int>::iterator it = myvector.begin(); it != myvector.end(); ++it)
         std::cout << *it << ' ';
     
     结果:
     第一次排序:
     1 2 3 4 5 6 9 7
     第二次排序:
     9 7 6 5 1 2 3 4
     ```

   - `partial_sort_copy()`排序函数

     ```cpp
     class mycomp2 {
     public:
         bool operator() (int i, int j) {
             return (i > j);
         }
     };
     
     int myints[5] = { 0 };
     std::list<int> mylist{ 3,2,5,4,1,6,9,7 };
     //按照默认的排序规则进行部分排序
     std::partial_sort_copy(mylist.begin(), mylist.end(), myints, myints + 5);
     cout << "第一次排序：\n";
     for (int i = 0; i < 5; i++) {
         cout << myints[i] << " ";
     }
     
     //以自定义的 mycomp2 作为排序规则，进行部分排序
     std::partial_sort_copy(mylist.begin(), mylist.end(), myints, myints + 5, mycomp2());
     cout << "\n第二次排序：\n";
     for (int i = 0; i < 5; i++) {
         cout << myints[i] << " ";
     }
     
     结果:
     第一次排序:
     1 2 3 4 5 6 9 7
     第二次排序:
     9 7 6 5 1 2 3 4
     ```

   - `nth_element`:  **当采用默认的升序排序规则(`std::less<T>`)时,该函数可以从某个序列中找到第`n`小的元素`K`，并将`K`移动到序列中第`n`的位置处。不仅如此，整个序列经过`nth_element()`函数处理后，所有位于`K`之前的元素都比`K`小，所有位于`K`之后的元素都比 K 大**

     ```cpp
     std::vector<int> _v01 = {3, 4, 1, 2, 5};
     std::nth_element(_v01.begin(), _v01.begin() + 2, _v01.end());
     // 默认从小到大,_v01中最终保存的结果 {2,1,3,4,5}
     
     class mycom2 {
      public:
       bool operator()(const int &a, const int &b) const { return a > b; }
     };
     
     std::vector<int> _v01 = {3, 4, 1, 2, 5};
     std::nth_element(_v01.begin(), _v01.begin() + 2, _v01.end(), mycom2());
     // 从大到小,_v01中最终保存的结果 {4,5,3,2,1}
     ```
     
   - **`partition`: 将标准容器中的元素按照 是否满足某个条件区分开。返回值是 满足条件的前半部分的下一个位置。当然`partition`还可以结合`erease()`来使用，将不满足条件的数据全部删除**

     ```cpp
     class mycom2 {
      public:
       bool operator()(const int &a) const { return (a % 2 == 1); }
     };
     
     std::vector<int> _v01 = {1, 2, 3, 4, 5, 6, 7, 8, 9};
     auto retIt = std::partition(_v01.begin(), _v01.end(), mycom2());
     
     std::cout << "[";
     for (auto &item : _v01) std::cout << item << ",";
     std::cout << "]" << std::endl;
     打印结果:
     [1,9,3,7,5,6,4,8,2,]
     
     std::cout << "[";
     for (auto it01 = _v01.begin(); it01 != retIt; it01++)
       std::cout << *it01 << ",";
     std::cout << "]" << std::endl;
     打印结果:
     [1,9,3,7,5,]
     ```
     
   - `statble_partition()`、`partition_copy()`

   - **`bool binary_search(first,last,const T& val)`:有序的区间[first,last)进行二分查找。如果找到则返回true,否则返回false**

     ```cpp
     std::vector<int> _v01 = {1, 2, 3, 4, 5, 6, 7, 8, 9};
     bool hasEle3 = std::binary_search(_v01.begin(), _v01.end(), 3);
     std::cout << "hasEle3==" << hasEle3 << std::endl;
     ```

   - **lower_bound(): 用于在指定区域内查找不小于目标值的第一个元素。函数返回一个正向迭代器,查找成功时, 迭代器指向找到的元素;反之,如果查找失败,迭代器指向和last迭代器相同。**

     ```cpp
     std::vector<int> _v01 = {1, 2, 3, 4, 5, 6, 7, 8, 9};
     auto iterRet = std::lower_bound(_v01.begin(), _v01.end(), 7);
     std::cout << "iterRet==" << *iterRet << std::endl;
     结果: 7
     ```

   - **upper_bound():用于在指定区域内查找大于目标值的第一个元素。函数返回一个正向迭代器,查找成功时, 迭代器指向找到的元素;反之,如果查找失败,迭代器指向和last迭代器相同。**

     ```cpp
     //以函数对象的形式定义查找规则(倒序)
     class mycomp2 {
     public:
         bool operator()(const int& i, const int& j) {
             return i > j;
         }
     };
     
     int a[5] = { 1,2,3,4,5 };
     //从 a 数组中找到第一个大于 3 的元素
     int *p = upper_bound(a, a + 5, 3);
     cout << "*p = " << *p << endl;
     
     vector<int> myvector{ 4,5,3,1,2 };
     //根据 mycomp2 规则，从 myvector 容器中找到第一个违背 mycomp2 规则的元素
     vector<int>::iterator iter = upper_bound(myvector.begin(), myvector.end(), 3, mycomp2());
     cout << "*iter = " << *iter;
     
     输出结果为:
     *p = 4
     *iter = 1
     ```

   - **equal_range(): 用于在指定区域内查找等于目标值的元素。返回值是一个pair, pair中 第一个迭代器指向的是 第一个等于 目标值的元素；第二个迭代器指向 第一个大于val的元素。如果查找失败，则两个迭代器要么都指向大于val的第一个元素，要么都和last迭代器相同**

     ```cpp
     //以函数对象的形式定义查找规则
     class mycomp2 {
     public:
         bool operator()(const int& i, const int& j) {
             return i > j;
         }
     };
     int main() {
         int a[9] = { 1,2,3,4,4,4,5,6,7};
         //从 a 数组中找到所有的元素 4
         pair<int*, int*> range = equal_range(a, a + 9, 4);
         cout << "a[9]：";
         for (int *p = range.first; p < range.second; ++p) {
             cout << *p << " ";
         }
     
         vector<int>myvector{ 7,8,5,4,3,3,3,3,2,1 };
         pair<vector<int>::iterator, vector<int>::iterator> range2;
         //在 myvector 容器中找到所有的元素 3
         range2 = equal_range(myvector.begin(), myvector.end(), 3,mycomp2());
         cout << "\nmyvector：";
         for (auto it = range2.first; it != range2.second; ++it) {
             cout << *it << " ";
         }
         return 0;
     }
     ```

   - **merge():将两个有序容器合并为一个有序容器，这两个容器的排序规则相同(要么升序，要么降序)**

     ```cpp
     //first 和 second 数组中各存有 1 个有序序列
     int first[] = { 5,10,15,20,25 };
     int second[] = { 7,17,27,37,47,57 };
     //用于存储新的有序序列
     std::vector<int> myvector(11);
     //将 [first,first+5) 和 [second,second+6) 合并为 1 个有序序列，并存储到 myvector 容器中。
     std::merge(first, first + 5, second, second + 6, myvector.begin());
     //输出 myvector 容器中存储的元素
     for (auto it = myvector.begin(); it != myvector.end(); ++it) {
         cout << *it << ' ';
     }
     ```

   - **inplace_merge():当 2 个有序序列存储在同一个数组或容器中时，如果想将它们合并为 1 个有序序列，除了使用 merge() 函数，更推荐使用 inplace_merge() 函数**

     ```cpp
     //该数组中存储有 2 个有序序列
     int first[] = { 5,10,15,20,25,7,17,27,37,47,57 };
     //将 [first,first+5) 和 [first+5,first+11) 合并为 1 个有序序列。
     inplace_merge(first, first + 5,first +11);
     
     for (int i = 0; i < 11; i++) {
         cout << first[i] << " ";
     }
     ```

   - **copy()/copy_n():从源容器中复制指定个数的元素到目的容器中**

     - 第一个参数是指向第一个源元素的迭代器；
     - 第二个参数是需要复制的个数;
     - 第三个参数是指向目的容器的第一个位置的迭代器;
     - 算法返回 指向最后一个被复制元素的下一个位置 的迭代器

     ```cpp
     std::vector<int> u1 = {2,6,8,4,9,4};
     std::vector<int> u2(6); //先分配了足够的空间
     std::copy(u1.begin(), u1.begin()+3, u2.begin());
     std::cout<<"The new vector with copy contains:";
     for(int k=0; k<u2.size(); k++)
     std::cout<<u2[k]<<" ";
     
     输出结果: The new vector with copy contains: 2 6 8 0 0 0
     
     int newints[]={15,25,35,45,55,65,75};
     std::vector<int> newvector;
     newvector.resize(7);//先分配了足够的空间
     std::copy_n(newints,7,newvector.begin());
     //最后newvector中保存的的数据15 25 35 45 55 65 75
     
     std::vector<int> _v01 = {1, 2, 3, 4, 5, 6, 7, 8, 9};
     std::set<int> more_nums{100, 112};
     std::copy_n(std::rbegin(_v01) + 1, 3,std::inserter(more_nums, std::begin(more_nums)));
     //最后more_nums中保存的数据 6 7 8 100 112
     ```

   - **copy_if():从源容器中复制满足条件的元素到 目的容器中**

     - 前两个参数定义源容器中的输入迭代器 范围;
     - 第三个参数是 指向目的容器的第一个位置的输出迭代器;
     - 第四个参数是 条件;

     ```cpp
     std::vector<std::string> names {"A1", "Beth", "Carol", "Dan", "Eve","Fred", "George", "Harry", "Iain", "Joe"};
     std::unordered_set<std::string> more_names {"Jean", "John"};
     size_t max_length{4};
     std::copy_if(std::begin(names), std::end(names), std::inserter(more_names, std::begin(more_names)), [max_length](const string& s) { return s.length() <= max_length;});
     //最后more_names中保存的数据 A1,Beth,Dan,Eve,Fred,Iain,Jean,Joe,John
     ```

   - **transform()有两种不同的使用方式**

     - 原文文档: https://vimsky.com/zh-tw/examples/usage/cpp-algorithm-transform-function-01.html

     - 一元运算: 该方法对`[first,last)`范围内的元素执行一元运算，并将结果存储在目标容器中

          ![image-20220407002641039](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220407002641039.png)

     - 二元运算: 该方法对`[first1,last1)`范围内的元素进行二元运算。这个`transform()`需要两个范围,并在输入范围的每对元素上应用一个带有2个参数的函数;

          ![image-20220407002915284](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220407002915284.png)

          示例一:

          ```cpp
          std::vector<int> v = { 3,1,4 };
          std::vector<std::string> result;
          
          std::transform(v.begin(), v.end(), std::back_inserter(result), [](int x) { return std::to_string(x * 2); });
          结果: 6 2 8
          
          std::vector<int> arr{ 1, 3, 5 };
          std::vector<int> arr2{ 1, 3, 5 };
          std::vector<int> arr3{ 1, 3, 5 };
          
          std::transform(arr.begin(), arr.end(), arr.begin(),[](int d) -> int {return d * 5; }); // // 5 15 25
          
          std::for_each(arr2.begin(), arr2.end(), [](int& a) {a *= 5; }); // 5 15 25
          
          // 5 15 25
          for (auto& value : arr3) {
            value *= 5;
          }
          ```

          示例二:

          ```cpp
          int op_increase (int i) { return ++i; }
          
          int main () {
            vector<int> foo;
            vector<int> bar;
          
            // set some values:
            for (int i=1; i<6; i++)
              foo.push_back (i*10);                         // foo:10 20 30 40 50
          
            bar.resize(foo.size());                         // allocate space
          
            transform (foo.begin(), foo.end(), bar.begin(), op_increase);
                                                            // bar:11 21 31 41 51
          
            // plus adds together its two arguments:
            transform (foo.begin(), foo.end(), bar.begin(), foo.begin(), plus<int>());
                                                            // foo:21 41 61 81 101
          
            cout << "foo contains:";
            for (vector<int>::iterator it=foo.begin(); it!=foo.end(); ++it)
              cout << ' ' << *it;
            cout << '\n';
          
            return 0;
          }
          输出 foo contains:21 41 61 81 101
          ```

   - **std::inserter: 构造一个插入迭代器,该插入迭代器在从it指向的位置开始的连续位置, 将新元素插入到x中。**

     > std::inserter(Container& x, typename Container::iterator it);
     > x:Container in which new elements will be inserted.
     > it:Iterator pointing to the insertion point.
     >
     > 返回：An insert_iterator that inserts elements into x at the position indicated by it.

     示例一:

     ```cpp
     // Declaring first container 
     deque<int> v1 = { 1, 2, 3 }; 
     
     // Declaring second container for 
     // copying values 
     deque<int> v2 = { 4, 5, 6 }; 
     
     deque<int>::iterator i1; 
     i1 = v2.begin() + 1; 
     // i1 points to next element of 4 in v2 
     
     // Using std::inserter inside std::copy 
     std::copy(v1.begin(), v1.end(), std::inserter(v2, i1)); 
     // v2 now contains 4 1 2 3 5 6
     ```

     示例二:

     ```cpp
     std::vector<std::string> names {"A1", "Beth", "Carol", "Dan", "Eve","Fred", "George", "Harry", "Iain", "Joe"};
     std::unordered_set<std::string> more_names {"Jean", "John"};
     size_t max_length{4};
     std::copy_if(std::begin(names), std::end(names), std::inserter(more_names, std::begin(more_names)), [max_length](const string& s) { return s.length() <= max_length;});
     //最后more_names中保存的数据 A1,Beth,Dan,Eve,Fred,Iain,Jean,Joe,John
     ```

     - `std::inserter`应该是通过`std::instert()`来完成的，所以容器必须支持`std::insert()`才能调用 `std::inserter`。也就是vector、list、deque等才能使用;

   - **std::back_inserter 用于在容器末尾插入元素。可以使用`back_inserter`的容器必须支持`push_back`成员函数，所以只有vector、deque、list等才能使用**

     示例一:

     ```cpp
     std::vector<int> v1{ 1, 2, 3, 4, 5, 6 };
     *(std::back_inserter(v1)) = 10;
     // v1 中的数据 1,2,3,4,5,6,10
     ```

     示例二:

     ```cpp
     std::vector<int> v1{ 1, 2, 3, 4, 5, 6};
     std::vector<int> v2{ 10, 20, 30, 40, 50, 60};
     std::copy(v2.begin(), v2.end(), std::back_inserter(v1));
     //v1 中的数据 1, 2, 3, 4, 5, 6, 10, 20, 30, 40, 50, 60
     ```

14. **容器的成员函数优先于同名的算法**

    a) **关联容器提供了count、find、lower_bound、upper_bound、equal_range**

    b) **list提供了remove、remove_if、unique、sort、merge、reverse**

    c) 有两个原因:

    i 成员函数通常与容器(特别是关联容器)结合更加紧密

    ii 成员函数往往速度更快;

15. **正确区分`count`、`find`、`binary_search`、`lower_bound`、`upper_bound`、`equal_range`**

    ![图片](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/640.png)