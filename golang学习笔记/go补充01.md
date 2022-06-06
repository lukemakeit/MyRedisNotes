### 高性能的一些技巧

文章: [再不Go就来不及了！Go高性能编程技法解读](https://mp.weixin.qq.com/s/fXKSr8GXaYxG1WCrLIgneg)

1. 反射性能损耗较高，尽量减少使用;

2. 数字转为字符串，优先使用`strconv`而不是`fmt`。因为fmt中使用了反射;

3. **避免重复的字符串到字节切片的转换**

   ```go
   func BenchmarkStringToByte(b *testing.B) {
      for i := 0; i < b.N; i++ {
         by := []byte("Hello world")
         _ = by
      }
   }
   
   func BenchmarkStringToByteOnce(b *testing.B) {
      bys := []byte("Hello world")
      for i := 0; i < b.N; i++ {
         by := bys
         _ = by
      }
   }
   ```

4. 指定容器容量

   - 指定map的容量提示`make(map[T1]T2, hint)`

     向make()提供容量提示会在初始化时尝试调整map的大小，这将减少在将元素添加到map时为map重新分配内存。

     注意，与slice不同。map capacity提示并不保证完全的抢占式分配，而是用于估计所需的hashmap bucket的数量。因此，在将元素添加到map时，甚至在指定map容量时，仍可能发生分配。

     ```go
     // Bad
     m := make(map[string]os.FileInfo)
     
     files, _ := ioutil.ReadDir("./files")
     for _, f := range files {
         m[f.Name()] = f
     }
     // m 是在没有大小提示的情况下创建的；在运行时可能会有更多分配。
     
     // Good
     files, _ := ioutil.ReadDir("./files")
     
     m := make(map[string]os.FileInfo, len(files))
     for _, f := range files {
         m[f.Name()] = f
     }
     // m 是有大小提示创建的；在运行时可能会有更少的分配。
     ```

   - 指定切片容量`make([]T,length,capacity)`

     在使用make()初始化切片时提供容量信息，特别是在追加切片时。

     与map不同，slice capacity不是一个提示：编译器将为提供给make()的slice的容量分配足够的内存，这意味着后续的append() 操作将导致零分配（直到slice的长度与容量匹配，在此之后，任何append都可能调整大小以容纳其他元素）。

     ```go
     const size = 1000000
     // Bad
     for n := 0; n < b.N; n++ {
       data := make([]int, 0)
         for k := 0; k < size; k++ {
           data = append(data, k)
       }
     }
     BenchmarkBad-4    219    5202179 ns/op
     
     // Good
     for n := 0; n < b.N; n++ {
       data := make([]int, 0, size)
         for k := 0; k < size; k++ {
           data = append(data, k)
       }
     }
     BenchmarkGood-4   706    1528934 ns/op
     ```

5. 字符串拼接方式的选择

   - 行内拼接字符串推荐使用 运算符+ 
     - 运算符+ 只能简单的完成字符串之间的拼接，非字符串类型的变量需要单独做类型转换。行内拼接字符串不会产生内存分配，也不涉及类型地动态转换，所以性能上优于`fmt.Sprintf()`
     - `fmt.Sprintf()`能够接收不同类型的入参，通过格式化输出完成字符串的拼接，使用非常方便。但因其底层实现使用了反射，性能上会有所损耗
     - **<mark style="color:blue">从性能出发，兼顾易用可读，如果待拼接的变量不涉及类型转换且数量较少<=5），行内拼接字符串推荐使用运算符+，反之使用`fmt.Sprintf()`</mark>**

6. 遍历`[]struct{}`使用下标而不是range。使用`[]*struct{}`则用range没啥问题;

   因为range时获取的是值拷贝的副本，对副本的修改，是不会影响到原切片。性能不会好。

   ```go
   type Item struct {
     id  int
     val [1024]byte
   }
   func BenchmarkRange1(b *testing.B) {
     items := make([]Item, 1024)
     tmps := make([]int, 1024)
     for i := 0; i < b.N; i++ {
       for j := range items {
         tmps[j] = items[j].id
       }
     }
   }
   
   func BenchmarkRange2(b *testing.B) {
     items := make([]Item, 1024)
     tmps := make([]int, 1024)
     for i := 0; i < b.N; i++ {
       for j, item := range items { //range会多很多次拷贝操作
         tmps[j] = item.id
       }
     }
   }
   ```

7. 使用空结构体`struct{}`节省内存

   - `strcut{}`不占内存空间

     ```go
     package main
     
     import (
       "fmt"
       "unsafe"
     )
     
     func main() {
       fmt.Println(unsafe.Sizeof(struct{}{}))
     }
     结果:
     0
     ```

   - 使用`struct{}`实现集合`set`

     ```go
     type Set map[string]struct{}
     
     func (s Set) Has(key string) bool {
       _, ok := s[key]
       return ok
     }
     
     func (s Set) Add(key string) {
       s[key] = struct{}{}
     }
     
     func (s Set) Delete(key string) {
       delete(s, key)
     }
     ```

   - 不发送数据的channel

     ```go
     func worker(ch chan struct{}) {
       <-ch
       fmt.Println("do something")
     }
     
     func main() {
       ch := make(chan struct{})
       go worker(ch)
       ch <- struct{}{}
       close(ch)
     }
     ```

   - 仅包含方法的结构体

     ```go
     type Door struct{}
     
     func (d Door) Open() {
       fmt.Println("Open the door")
     }
     
     func (d Door) Close() {
       fmt.Println("Close the door")
     }
     ```

8. Struct 布局需要考虑内存对齐

   - **为什么需要内存对齐?**

     CPU访问内存时，并不是逐个字节访问，而是以字长（word size）为单位访问。比如32位的CPU，字长为4字节，那么CPU访问内存的单位也是4字节。

     这么设计的目的，是减少CPU访问内存的次数，加大CPU访问内存的吞吐量。比如同样读取8个字节的数据，一次读取4个字节那么只需要读取2次。

   - 合理的`struct`布局

     ```
     type demo1 struct {
       a int8
       b int16
       c int32
     }
     
     type demo2 struct {
       a int8
       c int32
       b int16
     }
     
     func main() {
       fmt.Println(unsafe.Sizeof(demo1{})) // 8
       fmt.Println(unsafe.Sizeof(demo2{})) // 12
     }
     ```

     **因此，在对内存特别敏感的结构体的设计上，我们可以通过调整字段的顺序，将字段宽度从小到大由上到下排列，来减少内存的占用。**

9. **<mark style="color:blue">sync.Pool复用对象</mark>**

参考文章: 

- [sync.Pool 的使用场景](https://geektutu.com/post/hpg-sync-pool.html)
- [使用 Sync Pool 提升程序性能](https://marksuper.xyz/2021/09/02/sync_pool/)

sync.Pool可以作为保存临时取还对象的一个"池子"。**不过Pool中装的对象可以被无通知地被回收。**

sync.Pool是可伸缩的，同时也是并发安全的，其容量仅受限于内存的大小。存放在池中的对象如果不活跃了会被自动清理。

- **作用**

  对于很多需要重复分配、回收内存的地方，sync.Pool是一个很好的选择。频繁地分配、回收内存会给GC带来一定的负担，严重的时候会引起CPU的毛刺，而sync.Pool可以将暂时不用的对象缓存起来，待下次需要的时候直接使用，不用再次经过内存分配，复用对象的内存，减轻GC的压力，提升系统的性能。

  一句话: **<mark style="color:blue">用来保存和复用临时对象,注意是临时对象，减少内存分配，降低GC压力。有点感觉像是一个`free_nodes`的链表，`free_nodes`中有空的node，则给你返回，没有了则给新建一个。`free_nodes`很大了自动回收。</mark>**

- sync.Pool 本身就是线程安全的，多个 goroutine 可以并发地调用它的方法存取对象;

- sync.Pool 不可在使用之后再复制使用;

- 如何使用

  `sync.Pool`的使用方式非常简单，只需要实现New函数`func() interface{}`即可。

  当调用 Pool 的 Get 方法从池中获取元素，没有更多的空闲元素可返回时，将会调用New函数创建。

  如果你没有设置 New 字段，没有更多的空闲元素可返回时，Get 方法将返回 nil，表明当前没有可用的元素。

  ```go
  type Student struct {
    Name   string
    Age    int32
    Remark [1024]byte
  }
  
  var studentPool = sync.Pool{
      New: func() interface{} { 
          return new(Student) 
      },
  }
  ```

  然后调用Pool的Get()和Put()方法来获取和放回池子中。

  ```go
  stu := studentPool.Get().(*Student)
  json.Unmarshal(buf, stu)
  studentPool.Put(stu)
  ```

  - `Get()`用于从对象池中获取对象，因为返回值是interface{}，因此需要类型转换。调用该方法就会从`Pool`中取走一个元素。

    就意味着**这个元素会从 Pool 中移除，返回给调用者**。不过，除了返回值是正常实例化的元素，Get 方法的返回值还可能会是一个 nil（Pool.New 字段没有设置，又没有空闲元素可以返回），所以你在使用的时候，可能需要判断。

  - `Put()`则是在对象使用完毕后，放回到`Pool`,该元素可以服用。如果Put一个nil值，Pool会直接忽略。

  ```go
  var pool *sync.Pool
  
  type Person struct {
  	Name string
  }
  
  func initPool() {
  	pool = &sync.Pool {
  		New: func()interface{} {
  			fmt.Println("Creating a new Person")
  			return new(Person)
  		},
  	}
  }
  
  func main() {
  	initPool()
  
  	p := pool.Get().(*Person)
  	fmt.Println("首次从 pool 里获取：", p)
  
  	p.Name = "first"
  	fmt.Printf("设置 p.Name = %s\n", p.Name)
  
  	pool.Put(p)
  
  	fmt.Println("Pool 里已有一个对象：&{first}，调用 Get: ", pool.Get().(*Person))
  	fmt.Println("Pool 没有对象了，调用 Get: ", pool.Get().(*Person))
  }
  结果:
  Creating a new Person
  首次从 pool 里获取： &{}
  设置 p.Name = first
  Pool 里已有一个对象：&{first}，调用 Get:  &{first}
  Creating a new Person
  Pool 没有对象了，调用 Get:  &{}
  ```

  **注意:  Get 方法取出来的对象和上次 Put 进去的对象实际上是同一个，Pool 没有做任何“清空”的处理。但我们不应当对此有任何假设，因为在实际的并发使用场景中，无法保证这种顺序，最好的做法是在 Put 前，将对象清空。**