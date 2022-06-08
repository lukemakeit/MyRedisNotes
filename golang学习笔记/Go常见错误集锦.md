## Go 常见错误集锦
### 字符串底层原理及常见错误
原文: [https://gocn.vip/topics/Go488nfroE](https://gocn.vip/topics/Go488nfroE)  

1. `len()`函数返回的是字符串所占用的字节个数,而不是字符个数  
```go
a := "中国"
fmt.Printf("a length=%d\n", len(a)) //结果是6
```
Go语言对源代码默认使用`utf-8`编码,`utf-8`对"中"使用3字节保存;

2. `rune`

unicode字符集是对世界上多种语言字符的通用编码，也叫万国码。在unicode字符集中，每一个字符都有一个对应的编号，我们称这个编号为code point，而Go中的**rune类型就代表一个字符的code point**  
字符集只是将每个字符给了一个唯一的编码而已。而要想在计算机中进行存储，则必须要通过特定的编码转换成对应的二进制才行。所以就有了像ASCII、UTF-8、UTF-16等这样的编码方式。而在Go中默认是使用UTF-8字符编码进行编码的。所有unicode字符集合和字符编码之间的关系如下图所示：
![20220531174655](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/20220531174655.png)
UTF-8字符编码是一种变长字节的编码方式，用1到4个字节对字符进行编码，即最多4个字节，按位表示就是32位。所以，在Go的源码中，我们会看到对rune的定义是int32的别名:
```go
// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32
```
3. `strings.TrimRight`和`strings.TrimSuffix`的区别  
- `strings.TrimRight(s,cutset string)`函数 
该函数的功能是：从s字符串的末尾依次查找每一个字符，如果该字符包含在cutset中，则被移除，直到遇到第一个不在cutset中的字符。  
```go
fmt.Println(strings.TrimRight("123abbc", "bac")) //结果为 123
```
- `strings.TrimSuffix(s,suffix string) string`  
函数功能: 从字符串s中截取末尾的长度和suffix字符串长度相等的子字符串，然后和suffix字符串进行比较，如果相等，则将s字符串末尾的子字符串移除，如果不等，则返回原来的s字符串，该函数只截取一次。  
```go
strings.TrimSuffix("123abab", "ab") //结果为 123ab
```
4. **字符串拼接性能**  
低性能拼接字符串(多次内存分配 和 数据拷贝):
```go
func concat(ids []string) string {
	s := ""
	for _, id := range ids {
		s += id
	}
	return s
}
```
高性能拼接字符串:  
- go 1.10以前使用`bytes.buffer`;
- go 1.10以后使用`strings.builder`,`stings.join()`就是使用`strings.builder()`来实现的;
```go
func concat(ids []string) string {
	sb := strings.Builder{} 
	for _, id := range ids {
		_, _ = sb.WriteString(id) 
	}
	return sb.String() 
}
```
**可以通过strings.Builder进行改进。strings.Builder本质上是分配了一个字节切片，然后通过append的操作，将字符串的字节依次加入到该字节切片中。因为切片预分配空间的特性，可参考切片扩容，以有效的减少内存分配的次数，以提高性能。**  
**还可以通过Builder的`Grow`方法来预分配内存。减少内存分配次数 并提高性能**  
```go
func concat(ids []string) string {
	total := 0
	for i := 0; i < len(ids); i++ { 
		total += len(ids[i])
	}
	
	sb := strings.Builder{}
	sb.Grow(total) 
	for _, id := range ids {
		_, _ = sb.WriteString(id)
	}
	return sb.String()
}
```
5. 子字符串操作以及内存泄漏
字符串的切分也会跟切片的切分一样，可能会造成内存泄露。  
下面这个例子中, log字符串的前4个字节保存着message的类型，我们现在需要提取出message类型。  
```go
func (s store) handleLog(log string) error {
	if len(log) < 4 {
		return errors.New("log is not correctly formatted")
	}
	msgType := log[:4]
	s.store(msgType)
	// Do something
}
```
通过`log[:4]`提取msgType有啥问题?  
假设参数log是一个包含成千上万个字符的字符串。当我们使用log[:4]操作时，实际上是返回了一个字节切片，该切片的长度是4，而容量则是log字符串的整体长度。那么实际上我们存储的message不是包含4个字节的空间，而是整个log字符串长度的空间。所以就有可能会造成内存泄露。 如下图所示:  
![20220531201848](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/20220531201848.png)
如何避免？使用拷贝。将msgType提取后拷贝到一个字节切片中，这时该字节切片的长度和容量都是4。
```go
func (s store) handleLog(log string) error {
 if len(log) < 36 {
 return errors.New("log is not correctly formatted")
 }
 msgType := string([]byte(log[:4])) 
 // Do something
}
```
#### Go语言的IO库那么多，我该怎么选?
[Go语言的IO库那么多，我该怎么选?](https://mp.weixin.qq.com/s/yxk7avyv7gY1XjuEI_0Lvw?version=4.0.6.99101&platform=mac) 

#### append操作slice时的副作用
原文:[https://gocn.vip/topics/qwjgzxH7oD](https://gocn.vip/topics/qwjgzxH7oD)  
- **对slice的切分实际上是作用在slice的底层数组上的;**  
- **对一个已存在的slice进行切分操作会创建一个新的slice,但都会指向相同的底层数组**。 因此，如果一个索引值对两个slice都可见，则使用更新一个slice时，如`s1[1]=10`，则更新会影响另外一个slice。
```go
s1 := []int{1, 2, 3}
s2 := s1[1:2]
s3 := append(s2, 10)
fmt.Printf("s1=%+v s2=%+v s3=%+v\n", s1, s2, s3)//结果: s1=[1 2 10] s2=[2] s3=[2 10]
``` 
三个切片共享一个底层数组，数据最后一个元素被更新为10。即使我们没有修改`s1[2]`，但`s1`的内容被真真切切的修改了。  
![20220531232003](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/20220531232003.png)  
用新切片作为函数参数传递时，是非常危险的:  
```go
s4 := []int{1, 2, 3}
f(s4[:2])
fmt.Printf("s4=%+v\n", s4) //结果: s4=[11 2 10]

func f(s []int) {
	s[0] = 11
	s = append(s, 10)
}
```

<mark><font color="red">**如何解决上述问题?**</font></mark>  
- **方案一:拷贝切片,通过对原切片拷贝,构建一个新切片变量**  
```go
func main() {
	s := []int{1, 2, 3}
	sCopy := make([]int, 2)
	copy(sCopy, s) ①
	
	f(sCopy)
	result := append(sCopy, s[2]) ②
	// Use result
}

func f(s []int) {
	// Update s
}
```
该方案的缺点就是需要对已存在的切片进行一次拷贝，如果切片很大，那拷贝时存储和性能就会成为问题。  
- **方案二: 限制切片容量,该方案是通过限制切片容量，在对切片进行操作时自动产生一个新的底层数据的方式来避免对原有切片副作用的产生(有点像copy-on-write)**  
满切片表达式:`s[low:high:max]`  和 普通切片表达式`s:[low:high]`的区别在于:
**`s[low:high:max]`的切片的容量是`max-low`，而`s[low:high]`的容量是s中底层数据的最大容量减去low**。 
```go
s1 := []int{1, 2, 3}
s2 := s1[1:2:2]
s3 := append(s2, 10)
fmt.Printf("s1=%+v s2=%+v s3=%+v\n", s1, s2, s3) //结果: s1=[1 2 3] s2=[2] s3=[2 10]


s4 := []int{1, 2, 3}
s5 := f(s4[:2:2])
fmt.Printf("s4=%+v s5=%+v\n", s4, s5) //结果: s4=[11 2 3] s5=[11 2 10]

func f(s []int) []int {
	s[0] = 11
	s = append(s, 10)
	return s
}
```
#### copy 函数对切片操作时容易犯的错误
错误示例:
```go
src := []int{0, 1, 2}
var dst []int
copy(dst, src) // dst依然为空[]
```
dst依然为空[],为什么?  
因为在使用copy函数时，copy是将两个切片变量中最小长度的元素个数拷贝到目的切片变量中。 用个公式表示应该会更简单点： 
拷贝到变量dst中的元素个数 = `min(len(dst), len(src))`  
该示例中也就是 0，因为dst的长度是0，src的长度是3。所以，copy只能拷贝0个元素。  
如果想拷贝一个完整的切片怎么办呢？那就需要目标切片的长度必须要大于或等于源切片的长度： 
```go
src := []int{0, 1, 2}
dst := make([]int, len(src)) ①
copy(dst, src)
fmt.Println(dst) // dst={0,1,2}
```
或者:
```go
s := []int{0, 1, 2, 3, 4}
copy(s[3:5], s) ①
fmt.Println(s) //s={0,1,2,0,1}
```