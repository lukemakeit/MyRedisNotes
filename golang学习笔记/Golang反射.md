### Go反射:reflect

文章:[Golang的反射reflect深入理解和示例](https://juejin.cn/post/6844903559335526407)

Golang类型设计的一些原则:

- 变量包括(`type`、`value`)两部分，所以`nil!=nil`是可能的

  `fmt.Println( (interface{})(nil) == (*int)(nil) ) // false`

- `type`包括`static type`和`concrete type`。`static type`就是我们平时在编码中看到的类型,如int、float等。`concrete type`是runtime时看到的类型;

- 断言是否成功，取决于`concrete type`，而不是`static type`。所以一个reader变量如果其`concrete type`实现了`write`方法，则它也可以被断言为`writer`。

- Golang的实现中，每个`interface`变量都有一对`pair`,`pair`中记录着变量的`(value,type)`。且pair在接口连续赋值的过程中是不变的

```go
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)

var r io.Reader
r = tty

var w io.Writer
w = r.(io.Writer)
```

#### reflect基本功能`TypeOf`和`ValueOf` 

```go
// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i.  ValueOf(nil) returns the zero 
func ValueOf(i interface{}) Value {...}

翻译一下：ValueOf用来获取输入参数接口中的数据的值，如果接口为空则返回0

// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {...}

翻译一下：TypeOf用来动态获取输入参数接口中的值的类型，如果接口为空则返回nil
```

```go
func main() {
	var num float64 = 1.2345

	fmt.Println("type: ", reflect.TypeOf(num))
	fmt.Println("value: ", reflect.ValueOf(num))
}

运行结果:
type:  float64
value:  1.2345
```

#### 已知原类型 强制类型转换

`realValue := value.Interface().(已知的类型)`

```go
func main() {
	var num float64 = 1.2345

	pointer := reflect.ValueOf(&num)
	value := reflect.ValueOf(num)

	// 可以理解为“强制转换”，但是需要注意的时候，转换的时候，如果转换的类型不完全符合，则直接panic
	// Golang 对类型要求非常严格，类型一定要完全符合
	// 如下两个，一个是*float64，一个是float64，如果弄混，则会panic
	convertPointer := pointer.Interface().(*float64)
	convertValue := value.Interface().(float64)

	fmt.Println(convertPointer)
	fmt.Println(convertValue)
}

运行结果：
0xc42000e238
1.2345
```

- **转换的时候，如果转换的类型不完全符合，则直接panic，类型要求非常严格**
- **转换的时候，要区分是指针还是指**

#### 未知原类型

```go
type User struct {
	Id   int
	Name string
	Age  int
}
func (u User) ReflectCallFunc() {
	fmt.Println("Allen.Wu ReflectCallFunc")
}

func main() {
	user := User{1, "Allen.Wu", 25}
	DoFiledAndMethod(user)
}

// 通过接口来获取任意参数，然后一一揭晓
func DoFiledAndMethod(input interface{}) {
	getType := reflect.TypeOf(input)
	fmt.Println("get Type is :", getType.Name())

	getValue := reflect.ValueOf(input)
	fmt.Println("get all Fields is:", getValue)

	// 获取方法字段
	// 1. 先获取interface的reflect.Type，然后通过NumField进行遍历
	// 2. 再通过reflect.Type的Field获取其Field
	// 3. 最后通过Field的Interface()得到对应的value
	for i := 0; i < getType.NumField(); i++ {
		field := getType.Field(i)
		value := getValue.Field(i).Interface()
		fmt.Printf("%s: %v = %v\n", field.Name, field.Type, value)
	}

	// 获取方法
	// 1. 先获取interface的reflect.Type，然后通过.NumMethod进行遍历
	for i := 0; i < getType.NumMethod(); i++ {
		m := getType.Method(i) //真实函数
		fmt.Printf("%s: %v\n", m.Name, m.Type)
	}
}
运行结果：
get Type is : User
get all Fields is: {1 Allen.Wu 25}
Id: int = 1
Name: string = Allen.Wu
Age: int = 25
ReflectCallFunc: func(main.User)
```

获取`interface{}`的具体类型以及属性的步骤为:

1. 先通过`reflect.TypeOf()`获取`interface{}`的`reflect.Type`，然后通过`NumField`进行遍历;
2. 再通过`reflect.Type`的`Field()`获取其`Field`;
3. 最后通过`Field.Name`获取属性名、`Field.Type`获取属性类型、`Field.Interface()`获取value;

获取`intereface{}`所属方法的步骤:

1. 先通过`reflect.TypeOf()`获取`interface{}`的`reflect.Type`,然后通过`NumMethod()`进行遍历;
2. 而后通过`reflect.Type`的`Method()`获取对应的真实方法(函数);
3. 而后通过`method.Name()`、`method.Type()`得知具体的方法名;

#### 通过reflect.Value类型设置实际变量的值

`reflect.Value`是通过`reflect.ValueOf(X)`获得的，只有当X是指针的时候，才可以通过`reflect.Value`修改实际变量X的值，即：要修改反射类型的对象就一定要保证其值是`addressable`的。

```go
func main() {
	var num float64 = 1.2345
	fmt.Println("old value of pointer:", num)

	// 通过reflect.ValueOf获取num中的reflect.Value，注意，参数必须是指针才能修改其值
	pointer := reflect.ValueOf(&num)
	newValue := pointer.Elem()

	fmt.Println("type of pointer:", newValue.Type())
	fmt.Println("settability of pointer:", newValue.CanSet())

	// 重新赋值
	newValue.SetFloat(77)
	fmt.Println("new value of pointer:", num)

	// 如果reflect.ValueOf的参数不是指针，会如何？
	pointer = reflect.ValueOf(num)
	//newValue = pointer.Elem() // 如果非指针，这里直接panic，“panic: reflect: call of reflect.Value.Elem on float64 Value”
}

运行结果：
old value of pointer: 1.2345
type of pointer: float64
settability of pointer: true
new value of pointer: 77
```

#### 通过`reflect.ValueOf()`来进行方法的调用

```go
type User struct {
	Id   int
	Name string
	Age  int
}

func (u User) ReflectCallFuncHasArgs(name string, age int) {
	fmt.Println("ReflectCallFuncHasArgs name: ", name, ", age:", age, "and origal User.Name:", u.Name)
}

func (u User) ReflectCallFuncNoArgs() {
	fmt.Println("ReflectCallFuncNoArgs")
}

// 如何通过反射来进行方法的调用？
// 本来可以用u.ReflectCallFuncXXX直接调用的，但是如果要通过反射，那么首先要将方法注册，也就是MethodByName，然后通过反射调动mv.Call

func main() {
	user := User{1, "Allen.Wu", 25}
	
	// 1. 要通过反射来调用起对应的方法，必须要先通过reflect.ValueOf(interface)来获取到reflect.Value，得到“反射类型对象”后才能做下一步处理
	getValue := reflect.ValueOf(user)

	// 一定要指定参数为正确的方法名
	// 2. 先看看带有参数的调用方法
	methodValue := getValue.MethodByName("ReflectCallFuncHasArgs")
	args := []reflect.Value{reflect.ValueOf("wudebao"), reflect.ValueOf(30)}
	methodValue.Call(args)

	// 一定要指定参数为正确的方法名
	// 3. 再看看无参数的调用方法
	methodValue = getValue.MethodByName("ReflectCallFuncNoArgs")
	args = make([]reflect.Value, 0)
	methodValue.Call(args)
}


运行结果：
ReflectCallFuncHasArgs name:  wudebao , age: 30 and origal User.Name: Allen.Wu
ReflectCallFuncNoArgs
```

#### 补充

- **对指针类型执行 reflect.Value**

  错误信息: `reflect: call of reflect.Value.FieldByName on ptr Value`

  参考问题: [reflect: call of reflect.Value.FieldByName on ptr Value](https://stackoverflow.com/questions/50098624/reflect-call-of-reflect-value-fieldbyname-on-ptr-value)

  示例一:

  ```go
  type Family struct {
     first string
     last string
  }
  type Person struct {
     name string
     family *Family
  }
  func Check(data interface{}) {
      var v = reflect.ValueOf(data)
  
      if v.Kind() == reflect.Struct {
          fmt.Println("was a struct")
          familyPtr := v.FieldByName("family")
          v = reflect.Indirect(familyPtr).FieldByName("last") //特别注意这里
          fmt.Println(v)
      }
  }
  
  func main(){
     per1 := Person{name:"niki",family:&Familys{first:"yam",last:"bari"}}
     Check(per1)
  }
  ```

  示例二:

  ```go
  //表的struct, 根据field names获取对应的values
  func (task *TbTendisDTSTask) GetFieldsValue(fieldNames []string) (ret []interface{}, err error) {
  	//最好,先判断field是否存在
  	ret = make([]interface{}, 0, len(fieldNames))
  	getValue := reflect.ValueOf(task)
  	for _, field01 := range fieldNames {
  		val01 := reflect.Indirect(getValue).FieldByName(field01) //特别注意这里
  		ret = append(ret, val01.Interface())
  	}
  	for i := 0; i < len(fieldNames); i++ {
  		fmt.Printf("fieldName:%s value:%v\n", fieldNames[i], ret[i])
  	}
  	return
  }
  ```

  
