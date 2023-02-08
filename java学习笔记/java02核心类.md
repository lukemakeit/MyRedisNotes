## 核心类

#### 字符串String

- **Java字符串是不可变的**，这种不可变是通过内部的`private final char[]`实现的

- 比较字符串相等必须用`==` 而不是`equals`

  ```java
  String s1="hello";
  String s2="HELLO".toLowerCase();
  System.out.println(s1==s2); //输出false
  System.out.println(s1.equals(s2)); //输出true
  ```

- 忽略大小写比较:`equalsIgnoreCase()`

- 其他的一些例子:
  ```java
  "Hello".contains("ll"); // true
  "Hello".indexOf("l"); // 2
  "Hello".lastIndexOf("l"); // 3
  "Hello".startsWith("He"); // true
  "Hello".endsWith("lo"); // true
  "  \tHello\r\n ".trim(); // "Hello"
  ```

  注意：`trim()`并没有改变字符串的内容，而是返回了一个新字符串。
  另一个`strip()`方法也可以移除字符串首尾空白字符。它和`trim()`不同的是，类似中文的空格字符`\u3000`也会被移除:

  ```java
  "\u3000Hello\u3000".strip(); // "Hello"
  " Hello ".stripLeading(); // "Hello "
  " Hello ".stripTrailing(); // " Hello"
  ```

  `String`还提供了`isEmpty()`和`isBlank()`来判断字符串是否为空和空白字符串:

  ```java
  "".isEmpty(); // true，因为字符串长度为0
  "  ".isEmpty(); // false，因为字符串长度不为0
  "  \n".isBlank(); // true，因为只包含空白字符
  " Hello ".isBlank(); // false，因为包含非空白字符
  ```

- 替换字符串:

  ```java
  String s = "hello";
  s.replace('l', 'w'); // "hewwo"，所有字符'l'被替换为'w'
  s.replace("ll", "~~"); // "he~~o"，所有子串"ll"被替换为"~~"
  ```

  另一种是通过正则表达式替换:
  ```java
  String s = "A,,B;C ,D";
  s.replaceAll("[\\,\\;\\s]+", ","); // "A,B,C,D"
  ```

- 分割字符串`split`
  ```java
  String s = "A,B,C,D";
  String[] ss = s.split("\\,"); // {"A", "B", "C", "D"}
  ```

- 拼接字符串`join`
  ```java
  String[] arr = {"A", "B", "C"};
  String s = String.join("***", arr); // "A***B***C"
  ```

- 格式化字符串
  ```java
  String s = "Hi %s, your score is %d!";
  System.out.println(s.formatted("Alice", 80));
  System.out.println(String.format("Hi %s, your score is %.2f!", "Bob", 59.5));
  ```

- 类型转换:

  ```java
  String.valueOf(123); // "123"
  String.valueOf(45.67); // "45.67"
  String.valueOf(true); // "true"
  String.valueOf(new Object()); // 类似java.lang.Object@636be97c
  
  int n1 = Integer.parseInt("123"); // 123
  int n2 = Integer.parseInt("ff", 16); // 按十六进制转换，255
  
  boolean b1 = Boolean.parseBoolean("true"); // true
  boolean b2 = Boolean.parseBoolean("FALSE"); // false
  
  //转换为char[]
  char[] cs = "Hello".toCharArray(); // String -> char[]
  String s = new String(cs); // char[] -> String
  ```

- **StringBuilder**

  Java编译器对`String`做了特殊处理，使得我们可以直接用`+`拼接字符串。
  ```java
  String s = "";
  for (int i = 0; i < 1000; i++) {
      s = s + "," + i;
  }
  ```

  虽然可以直接拼接字符串，但是，在循环中，每次循环都会创建新的字符串对象，然后扔掉旧的字符串。这样，绝大部分字符串都是临时对象，不但浪费内存，还会影响GC效率。
  为了能高效拼接字符串，**Java标准库提供了`StringBuilder`，它是一个可变对象，可以预分配缓冲区，这样，往`StringBuilder`中新增字符时，不会创建新的临时对象**：

  ```java
  StringBuilder sb = new StringBuilder(1024);
  for (int i = 0; i < 1000; i++) {
      sb.append(',');
      sb.append(i);
  }
  String s = sb.toString();
  
  //链式操作，append()方法会返回this
  var sb = new StringBuilder(1024);
  sb.append("Mr ")
  .append("Bob")
  .append("!")
  ```

  参考`StringBuilder`的链式设计，我们也可以设计一个类。
  ```java
  public class Main {
      public static void main(String[] args) {
          Adder adder = new Adder();
          adder.add(3)
               .add(5)
               .inc()
               .add(10);
          System.out.println(adder.value());
      }
  }
  class Adder {
      private int sum = 0;
      public Adder add(int n) {
          sum += n;
          return this;
      }
      public Adder inc() {
          sum ++;
          return this;
      }
      public int value() {
          return sum;
      }
  }
  ```

- **StringJoiner**
  先来看一个示例:

  ```java
  // Hello Bob, Alice, Grace!
  public class Main {
      public static void main(String[] args) {
          String[] names = {"Bob", "Alice", "Grace"};
          var sb = new StringBuilder();
          sb.append("Hello ");
          for (String name : names) {
              sb.append(name).append(", ");
          }
          // 注意去掉最后的", ":
          sb.delete(sb.length() - 2, sb.length());
          sb.append("!");
          System.out.println(sb.toString());
      }
  }
  ```

  StringJoiner 来实现同样的事情:
  ```java
  // Hello Bob, Alice, Grace!
  import java.util.StringJoiner;
  public class Main {
      public static void main(String[] args) {
          String[] names = {"Bob", "Alice", "Grace"};
          var sj = new StringJoiner(", ", "Hello ", "!"); // Hello 和 !是指定了头部和尾部
          for (String name : names) {
              sj.add(name);
          }
          System.out.println(sj.toString());
      }
  }
  ```

  静态方法**String.join()**内部使用`StringJoiner`来拼接字符串。
  ```java
  String[] names = {"Bob", "Alice", "Grace"};
  var s = String.join(", ", names);
  ```

#### 包装类型

Java数据类型:

- 基本类型: `byte`、`short`、`int`、`long`、`boolean`、`float`、`double`、`char`
- 引用类型: 所有的`class`和`interface`类型;

引用类型可以赋值为`null`，表示空，但基本类型不能赋值为`null`:
```java
String s = null;
int n = null; // compile error!
```

包装类型非常有用，Java核心库为每种基本类型都提供了对应的包装类型:

| 基本类型 | 对应的引用类型      |
| :------- | :------------------ |
| boolean  | java.lang.Boolean   |
| byte     | java.lang.Byte      |
| short    | java.lang.Short     |
| int      | java.lang.Integer   |
| long     | java.lang.Long      |
| float    | java.lang.Float     |
| double   | java.lang.Double    |
| char     | java.lang.Character |

```java
// 通过静态方法valueOf(int)创建Integer实例:
Integer n2 = Integer.valueOf(i);
// 通过静态方法valueOf(String)创建Integer实例:
Integer n3 = Integer.valueOf("100");
System.out.println(n3.intValue());
```

**Auto Boxing**
因为`int`和`Integer`可相互转换:

```java
int i = 100;
Integer n = Integer.valueOf(i);
int x = n.intValue();
```

Java编译器可以帮助我们自动在`int`和`Integer`之间转型:
```java
Integer n = 100; // 编译器自动使用Integer.valueOf(int)
int x = n; // 编译器自动使用Integer.intValue()
```

这种直接把`int`变为`Integer`的赋值写法，称为自动装箱（Auto Boxing），反过来，把`Integer`变为`int`的赋值写法，称为自动拆箱（Auto Unboxing）。注意：自动装箱和自动拆箱只发生在编译阶段，目的是为了少写代码。
**进制转换**
字符串转换为 int

```java
int x1 = Integer.parseInt("100"); // 100
int x2 = Integer.parseInt("100", 16); // 256,因为按16进制解析
```

`Integer`还可以把整数格式化为指定进制的字符串:

```java
System.out.println(Integer.toString(100)); // "100",表示为10进制
System.out.println(Integer.toString(100, 36)); // "2s",表示为36进制
System.out.println(Integer.toHexString(100)); // "64",表示为16进制
System.out.println(Integer.toOctalString(100)); // "144",表示为8进制
System.out.println(Integer.toBinaryString(100)); // "1100100",表示为2进制
```

一些有用的静态遍历:
```java
// boolean只有两个值true/false，其包装类型只需要引用Boolean提供的静态字段:
Boolean t = Boolean.TRUE;
Boolean f = Boolean.FALSE;
// int可表示的最大/最小值:
int max = Integer.MAX_VALUE; // 2147483647
int min = Integer.MIN_VALUE; // -2147483648
// long类型占用的bit和byte数量:
int sizeOfLong = Long.SIZE; // 64 (bits)
int bytesOfLong = Long.BYTES; // 8 (bytes)
```

**处理无符号整型**
在Java中，并没有无符号整型（Unsigned）的基本数据类型。`byte`、`short`、`int`和`long`都是带符号整型，最高位是符号位。

```java
byte x = -1;
byte y = 127;
System.out.println(Byte.toUnsignedInt(x)); // 255
System.out.println(Byte.toUnsignedInt(y)); // 127
```

