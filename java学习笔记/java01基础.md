### 环境搭建

- JDK(Java Development Kit, Java开发工具包)

  JDK是提供给java开发人员使用的，其中包含了java的开发工具包，也包括了JRE。安装JDK后，无需单独安装JRE了；

  其中开发工具: 编译工具(javac.exe) 打包工具(jar.exe)等;

- JRE(Java Runtime Environment,Java运行环境)

  包括Java虚拟机和java程序所需的核心类库等，如果想要运行一个开发好的java程序，计算机只需安装JRE即可。

- 简单而言，使用JDK的开发工具完成Java程序，交给JRE运行。

![image-20220814121401134](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220814121401134.png)

### 基础

- **在一个java源文件中可以声明多个class, 但是只能最多有一个类声明为public;**

  **而且要求public的类的类名必须和源文件名相同;**

- 程序的入口是main方法;

- 输出语句: `System.out.println("Hello world")`、`System.out.print()`

#### 变量

- **byte**

- **char**

- **boolean: 只能是true or false**

- 自动类型提升: byte、char、short -> int -> long -> float ->double

- 强制类型转换: 
  ```java
  double d1 = 12.3;
  int i1=(int)d1;
  
  long l1=123;
  short s2=(short)l1;
  ```

- **String**

#### 流程控制

```java
if(){
}else if(){
}else{
}

switch(){
  case 常量1:
    语句1;
    //break;
  case 常量2:
    语句2;
    //break
  default:
    语句;
    //break
}
没有break则继续朝下执行;

for(int i=0;i<100;i++){
  //break
  //continue
}

while(){
  
}

do
{
  
}while()
```

- **数组:Array**

  **一旦初始化完成后，其长度确定**

  ```java
  int[] idx; //声明
  ids = new int[]{0,1,2,3,4,5}; //静态初始化:数组的出书画和数组元素的赋值操作同时进行
  
  String[] names=new String[5]; //动态初始化: 数组的初始化和数组元素的赋值操作分开进行
  
  //错误写法
  int[] arr3=new int[3]{1,2,3};
  int[5] arr3=new int[5];
  
  //下标访问
  names[0]="张三";
  names[1]="李四";
  
  //数组长度
  names.length; // 5
  
  for(int i=0;i<names.length;i++){
    System.out.println(names[i]);
  }
  ```

- **java.util.Arrarys:操作数组的方法**

  ```java
  import java.util.Arrays;
  
  int[] arr1=new int[]{1,2,3,4};
  int[] arr2=new int[]{1,2,3,4};
  boolean isEquals=Arrays.equals(arr1, arr2);
  System.out.println(isEquals);
  
  Arrays.fill(arr1,10);
  System.out.println(Arrays.toString(arr1));//输出:[10,10,10,10]
  
  Arrays.sort(arr2); //排序
  ```

#### 函数

```java
public static void playGame(){
	System.out.println("开始打游戏");
}

public static int getSum(int num1,int num2,int num3){
  int ret =num1 + num2 + num3;
  return ret;
}

//重载
public static int sum(int a,int b){
  return a+b;
}
public static int sum(int a,int b,int c){
  return a+b+c;
}
```

#### 面向对象

```java
package com.atiguigu.contact;

public class Group {
    private String[] names;
    
    /*
     * 可通过如下方式赋值
     * g.setNames("Xiao Ming", "Xiao Hong");
     * g.setNames("Xiao Ming");
     * g.setNames();
    */
    public void setNames(String... names){
        this.names=names;
    }

    /*
     * 引用参数绑定机制
     * g.setNames(new String[] {"Xiao Ming", "Xiao Hong", "Xiao Jun"}); // 传入1个String[]
     * g.setNames(null);
    */
    public void setNames02(String[] names) {
        this.names=names;
    }

    public String getName(){
        return this.names[0] + " " + this.names[1];
    }
}

//外部 fullname 变了, g2.getName() 也变了
Group g2= new Group();
String[] fullname=new String[]{"Homer","Simpson"};
g2.setNames02(fullname);
System.out.println("before fullname==>"+g2.getName()); // 打印: before fullname==>Homer Simpson
fullname[0]="Bart";
System.out.println("after fullname==>"+g2.getName()); //打印: after fullname==>Bart Simpson
```

##### 构造函数:

```java
class Person {
    private String name; // 默认初始化为null
    private int age; // 默认初始化为0

    public Person() {
    }
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    public Person(String name) {
        this.name = name;
        this.age = 12;
    }
    public String getName() {
        return this.name;
    }
    public int getAge() {
        return this.age;
    }
}
```

##### 方法重载:

```java
class Hello {
    public void hello() {
        System.out.println("Hello, world!");
    }
    public void hello(String name) {
        System.out.println("Hello, " + name + "!");
    }
    public void hello(String name, int age) {
        if (age < 18) {
            System.out.println("Hi, " + name + "!");
        } else {
            System.out.println("Hello, " + name + "!");
        }
    }
}
```

##### 继承:

```java
class Person {
    private String name;
    private int age;

    public String getName() {...}
    public void setName(String name) {...}
    public int getAge() {...}
    public void setAge(int age) {...}
}

class Student extends Person {
    // 不要重复name和age字段/方法,
    // 只需要定义新增score字段/方法:
    private int score;

    public int getScore() { … }
    public void setScore(int score) { … }
}
```

**注意：子类自动获得了父类的所有字段，严禁定义与父类重名的字段！**
**在Java中，没有明确写`extends`的类，编译器会自动加上`extends Object`。所以，任何类，除了`Object`，都会继承自某个类。**

`Object -> Person -> Student`

**Java只允许一个class继承自一个类，因此，一个类有且仅有一个父类。只有`Object`特殊，它没有父类。**

##### **protected**

**继承有个特点，就是子类无法访问父类的`private`字段或者`private`方法。为了让子类可以访问父类的字段，我们需要把`private`改为`protected`**

```java
class Person {
    protected String name;
    protected int age;
}

class Student extends Person {
    public String hello() {
        return "Hello, " + name; // OK!
    }
}
```

##### **super**

`super`关键字表示父类（超类）。子类引用父类的字段时，可以用`super.fieldName`。例如:

```java
class Student extends Person {
    public String hello() {
        return "Hello, " + super.name;
    }
}
```

```java
class Person {
    protected String name;
    protected int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

class Student extends Person {
    protected int score;

    public Student(String name, int age, int score) {
        super(name,age); // 调用父类的构造方法Person(String, int)
        this.score = score;
    }
}
```

**因此我们得出结论：如果父类没有默认的构造方法，子类就必须显式调用`super()`并给出参数以便让编译器定位到父类的一个合适的构造方法。**

**向上转型**

```java
Student s = new Student();
Person p = s; // upcasting, ok
```

**向下转型**

```java
Person p1 = new Student(); // upcasting, ok
Person p2 = new Person();
Student s1 = (Student) p1; // ok
Student s2 = (Student) p2; // runtime error! ClassCastException!

// instanceof 很重要
Person p = new Student();
if (p instanceof Student) {
    // 只有判断成功才会向下转型:
    Student s = (Student) p; // 一定会成功
}
```

##### **多态**

子类如果定义了一个与父类方法签名完全相同的方法，被称为覆写(Override)。

```python
class Person {
    public void run() {
        System.out.println("Person.run");
    }
}

class Student extends Person {
    @Override
    public void run() {
        System.out.println("Student.run");
    }
}
```

Override和Overload不同的是，如果方法签名不同，就是Overload，Overload方法是一个新方法；如果方法签名相同，并且返回值也相同，就是`Override`。
**注意：方法名相同，方法参数相同，但方法返回值不同，也是不同的方法。在Java程序中，出现这种情况，编译器会报错。**
**加上`@Override`可以让编译器帮助检查是否进行了正确的覆写。希望进行覆写，但是不小心写错了方法签名，编译器会报错。**

```python
class Student extends Person {
    // 不是Override，因为参数不同:
    public void run(String s) { … }
    // 不是Override，因为返回值不同:
    public int run() { … }
}
```

一个更全面的示例:
```java
public class Main {
    public static void main(String[] args) {
        // 给一个有普通收入、工资收入和享受国务院特殊津贴的小伙伴算税:
        Income[] incomes = new Income[] {
            new Income(3000),
            new Salary(7500),
            new StateCouncilSpecialAllowance(15000)
        };
        System.out.println(totalTax(incomes));
    }

    public static double totalTax(Income... incomes) {
        double total = 0;
        for (Income income: incomes) {
            total = total + income.getTax();
        }
        return total;
    }
}
class Income {
    protected double income;

    public Income(double income) {
        this.income = income;
    }

    public double getTax() {
        return income * 0.1; // 一般性收入算,税率10%
    }
}
class Salary extends Income {
    public Salary(double income) {
        super(income);
    }
    @Override
    public double getTax() {
        if (income <= 5000) {
            return 0;
        }
        return (income - 5000) * 0.2; //工薪基层计算税率
    }
}
class StateCouncilSpecialAllowance extends Income {
    public StateCouncilSpecialAllowance(double income) {
        super(income);
    }
    @Override
    public double getTax() { //军人等不交税
        return 0;
    }
}
```

##### **复写Object方法:**
所有的class都继承自Object,而Object定义了几个重要的方法:

- **toString(): 把instance输出为String**
- **equals():判断两个instance是否逻辑相等**
- **hashCode():计算一个instance的哈希值**

```java
class Person {
    ...
    // 显示更有意义的字符串:
    @Override
    public String toString() {
        return "Person:name=" + name;
    }
    // 比较是否相等:
    @Override
    public boolean equals(Object o) {
        // 当且仅当o为Person类型:
        if (o instanceof Person) {
            Person p = (Person) o;
            // 并且name字段相同时，返回true:
            return this.name.equals(p.name);
        }
        return false;
    }
    // 计算hash:
    @Override
    public int hashCode() {
        return this.name.hashCode();
    }
}
```

##### **调用super**
在子类的覆写方法中，如果要调用父类的被覆写的方法，可以通过`super`来调用。例如:

```javascript
class Person {
    protected String name;
    public String hello() {
        return "Hello, " + name;
    }
}

Student extends Person {
    @Override
    public String hello() {
        // 调用父类的hello()方法:
        return super.hello() + "!";
    }
}
```

##### **final**

继承可以允许子类覆写父类的方法。如果一个父类不允许子类对它的某个方法进行覆写，可以把该方法标记为`final`。用`final`修饰的方法不能被`Override`:
```java
class Person {
    protected String name;
    public final String hello() {
        return "Hello, " + name;
    }
}
Student extends Person {
    // compile error: 不允许覆写
    @Override
    public String hello() {
    }
}
```

如果一个类不希望任何其他类继承自它，那么可以把这个类本身标记为`final`。用`final`修饰的类不能被继承:
```python
final class Person {
    protected String name;
}
// compile error: 不允许继承自Person
Student extends Person {
}
```

对于一个类的实例字段，同样可以用`final`修饰。用`final`修饰的字段在初始化后不能被修改。例如:
```java
class Person {
    public final String name = "Unamed";
}
Person p = new Person();
p.name = "New Name"; // compile error!
```

可以在构造方法中初始化final字段:
```java
class Person {
    public final String name;
    public Person(String name) {
        this.name = name;
    }
}
```

##### 抽象类

如果一个`class`定义了方法，但没有具体执行代码，这个方法就是抽象方法，抽象方法用`abstract`修饰。
因为无法执行抽象方法，因此这个类也必须申明为抽象类（abstract class）。
使用`abstract`修饰的类就是抽象类。我们无法实例化一个抽象类：

抽象类本身被设计成只能用于被继承，因此，抽象类可以强迫子类实现其定义的抽象方法，否则编译会报错。因此，抽象方法实际上相当于定义了“规范”。
```java
public class Main {
    public static void main(String[] args) {
        Person p = new Student();
        p.run();
    }
}
abstract class Person {
    public abstract void run();
}
class Student extends Person {
    @Override
    public void run() {
        System.out.println("Student.run");
    }
}
```

##### 接口

如果一个抽象类没有字段，所有方法全部都是抽象方法:
```python
abstract class Person {
    public abstract void run();
    public abstract String getName();
}
```

就可以把该抽象类改写为接口：`interface`。
```java
interface Person {
    void run();
    String getName();
}
```

所谓`interface`，就是比抽象类还要抽象的纯抽象接口，因为它连字段都不能有。因为接口定义的所有方法默认都是`public abstract`的，所以这两个修饰符不需要写出来（写不写效果都一样）。
当一个具体的`class`去实现一个`interface`时，需要使用`implements`关键字。举个例子:

```java
class Student implements Person {
    private String name;
    public Student(String name) {
        this.name = name;
    }
    @Override
    public void run() {
        System.out.println(this.name + " run");
    }
    @Override
    public String getName() {
        return this.name;
    }
}
```

我们知道，在Java中，一个类只能继承自另一个类，不能从多个类继承。但是，一个类可以实现多个`interface`，例如:
```java
class Student implements Person, Hello { // 实现了两个interface
    ...
}
```

**default方法**

```java
public class Main {
    public static void main(String[] args) {
        Person p = new Student("Xiao Ming");
        p.run();
    }
}
interface Person {
    String getName();
    default void run() {
        System.out.println(getName() + " run");
    }
}
class Student implements Person {
    private String name;
    public Student(String name) {
        this.name = name;
    }
    public String getName() {
        return this.name;
    }
}
```

**实现类可以不必覆写`default`方法。`default`方法的目的是，当我们需要给接口新增一个方法时，会涉及到修改全部子类。如果新增的是`default`方法，那么子类就不必全部修改，只需要在需要覆写的地方去覆写新增方法。（部分子类修改）**
`default`方法和抽象类的普通方法是有所不同的。因为`interface`没有字段，`default`方法无法访问字段，而抽象类的普通方法可以访问实例字段。

##### 静态字段和静态方法

- **静态字段**

- 实例字段在每个实例中都有自己的一个独立“空间”，但是静态字段只有一个共享“空间”，所有实例都会共享该字段。举个例子:

  ```java
  public class Main {
      public static void main(String[] args) {
          Person ming = new Person("Xiao Ming", 12);
          Person hong = new Person("Xiao Hong", 15);
          ming.number = 88;
          System.out.println(hong.number);
          hong.number = 99;
          System.out.println(ming.number);
      }
  }
  
  class Person {
      public String name;
      public int age;
      public static int number;
      public Person(String name, int age) {
          this.name = name;
          this.age = age;
      }
  }
  ```

  不推荐用`实例变量.静态字段`去访问静态字段，因为在Java程序中，实例对象并没有静态字段。

  推荐用类名来访问静态字段。
  ```java
  Person.number = 99;
  System.out.println(Person.number);
  ```

- **静态方法**

  调用实例方法必须通过一个实例变量，而调用静态方法则不需要实例变量，通过类名就可以调用。
  ```java
  public class Main {
      public static void main(String[] args) {
          Person.setNumber(99);
          System.out.println(Person.number);
      }
  }
  class Person {
      public static int number;
      public static void setNumber(int value) {
          number = value;
      }
  }
  ```

  - 因为静态方法属于`class`而不属于实例，因此，静态方法内部，无法访问`this`变量，也无法访问实例字段，它只能访问静态字段;
  - 静态方法也经常用于辅助方法。注意到Java程序的入口`main()`也是静态方法;

- **接口的静态字段**

  因为`interface`是一个纯抽象类，所以它不能定义实例字段。但是，`interface`是可以有静态字段的，并且静态字段必须为`final`类型:
  ```java
  public interface Person {
      public static final int MALE = 1;
      public static final int FEMALE = 2;
  }
  ```

  实际上，因为`interface`的字段只能是`public static final`类型，所以我们可以把这些修饰符都去掉，上述代码可以简写为:
  ```java
  public interface Person {
      // 编译器会自动加上public statc final:
      int MALE = 1;
      int FEMALE = 2;
  }
  ```

##### 包`package`

如 小明写了一个Person类，小红也写了一个 Person类，而我们两个Person类都想用。
小军写了一个Arrays类，恰好JDK也自带了一个Arrays类, 如何解决类名冲突?
**Java中通过package来解决名字冲突,小明的Person类存放在包ming下,完整类名是`ming.Person`。**
JDK的Arrays类存放在包`java.util`下面, 因此完整类名是`java.util.Arrays`。

- **import**

  ```java
  // 想引用别人的类,直接写出完整类名
  package ming;
  public class Person {
      public void run() {
          mr.jun.Arrays arrays = new mr.jun.Arrays();
      }
  }
  
  // Person.java
  package ming;
  // 导入完整类名:
  import mr.jun.Arrays;
  public class Person {
      public void run() {
          Arrays arrays = new Arrays();
      }
  }
  ```

- 在写`import`的时候，可以使用`*`，表示把这个包下面的所有`class`都导入进来（但不包括子包的`class`:
  我们一般不推荐这种写法，因为在导入了多个包后，很难看出`Arrays`类属于哪个包

  ```java
  // Person.java
  package ming;
  // 导入mr.jun包的所有class:
  import mr.jun.*;
  public class Person {
      public void run() {
          Arrays arrays = new Arrays();
      }
  }
  ```

  如果有两个`class`名称相同，例如，`mr.jun.Arrays`和`java.util.Arrays`，那么只能`import`其中一个，另一个必须写完整类名。

- **最佳实践**

  为了避免名字冲突，我们需要确定唯一的包名。推荐的做法是使用倒置的域名来确保唯一性。如:

  - org.apache
  - org.apache.commons.log
  - com.liaoxuefeng.sample

  要注意不要和`java.lang`包的类重名，即自己的类不要使用这些名字:

  - String
  - System
  - Runtime
  - ...

  ![image-20220815113434155](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220815113434155.png)

##### 作用域

- public : 可以被其他包、其他类访问
- protected: 可以被继承关系的子类访问
- private: 无法被其他类访问, 类的内部嵌套类可以访问 private;
- package: **一个类允许访问同一个`package`的没有`public`、`private`修饰的`class`，以及没有`public`、`protected`、`private`修饰的字段和方法**;

##### 内部类(Nested Class)

- inner Class

  ```java
  class Outer {
      class Inner {
          // 定义了一个Inner Class
      }
  }
  
  public class Main {
      public static void main(String[] args) {
          Outer outer = new Outer("Nested"); // 实例化一个Outer
          Outer.Inner inner = outer.new Inner(); // 实例化一个Inner
          inner.hello();
      }
  }
  class Outer {
      private String name;
      Outer(String name) {
          this.name = name;
      }
      class Inner {
          void hello() {
              System.out.println("Hello, " + Outer.this.name);
          }
      }
  }
  ```

  上述代码，要实例化一个`Inner`，我们必须首先创建一个`Outer`的实例，然后，调用`Outer`实例的`new`来创建`Inner`实例。

  Inner Class除了有一个`this`指向它自己，还隐含地持有一个Outer Class实例，可以用`Outer.this`访问这个实例。所以，实例化一个Inner Class不能脱离Outer实例。

  观察Java编译器编译后的`.class`文件可以发现，`Outer`类被编译为`Outer.class`，而`Inner`类被编译为`Outer$Inner.class`。

- Anonymous Class

  它不需要在Outer Class中明确地定义这个Class，而是在方法内部，通过匿名类（Anonymous Class）来定义:
  ```java
  public class Main {
      public static void main(String[] args) {
          Outer outer = new Outer("Nested");
          outer.asyncHello();
      }
  }
  
  class Outer {
      private String name;
      Outer(String name) {
          this.name = name;
      }
      void asyncHello() {
          Runnable r = new Runnable() {
              @Override
              public void run() {
                  System.out.println("Hello, " + Outer.this.name);
              }
          };
          new Thread(r).start();
      }
  }
  ```

  观察`asyncHello()`方法，我们在方法内部实例化了一个`Runnable`。`Runnable`本身是接口，接口是不能实例化的，所以这里实际上是定义了一个实现了`Runnable`接口的匿名类，并且通过`new`实例化该匿名类，然后转型为`Runnable`。在定义匿名类的时候就必须实例化它，定义匿名类的写法如下:
  ```java
  Runnable r = new Runnable() {
      // 实现必要的抽象方法...
  };
  ```

  匿名类和Inner Class一样，可以访问Outer Class的`private`字段和方法。之所以我们要定义匿名类，是因为在这里我们通常不关心类名，比直接定义Inner Class可以少写很多代码。
  观察Java编译器编译后的`.class`文件可以发现，`Outer`类被编译为`Outer.class`，而匿名类被编译为`Outer$1.class`。如果有多个匿名类，Java编译器会将每个匿名类依次命名为`Outer$1`、`Outer$2`、`Outer$3`……
  除了接口外，匿名类也完全可以继承自普通类。观察以下代码:

  ```java
  import java.util.HashMap;
  public class Main {
      public static void main(String[] args) {
          HashMap<String, String> map1 = new HashMap<>();
          HashMap<String, String> map2 = new HashMap<>() {}; // 匿名类!
          HashMap<String, String> map3 = new HashMap<>() {
              {
                  put("A", "1");
                  put("B", "2");
              }
          };
          System.out.println(map3.get("A"));
      }
  }
  ```

  `map1`是一个普通的`HashMap`实例，但`map2`是一个匿名类实例，只是该匿名类继承自`HashMap`。
  `map3`也是一个继承自`HashMap`的匿名类实例，并且添加了`static`代码块来初始化数据。观察编译输出可发现`Main$1.class`和`Main$2.class`两个匿名类文件。

##### classpath 和 jar

- `classpath`是JVM用到的一个环境变量，它用来指示JVM如何搜索`class`；

- 因为Java是编译型语言，源码文件是`.java`，而编译后的`.class`文件才是真正可以被JVM执行的字节码。因此，JVM需要知道，如果要加载一个`abc.xyz.Hello`的类，应该去哪搜索对应的`Hello.class`文件;

- `classpath`就是一组目录的集合，它设置的搜索路径与操作系统相关

  ```java
  // linux系统
  /usr/shared:/usr/local/bin:/home/liaoxuefeng/bin
    
  //windows 系统
  C:\work\project1\bin;C:\shared;"D:\My Documents\project1\bin"
  ```

  假设`classpath`是`.;C:\work\project1\bin;C:\shared`，当JVM在加载`abc.xyz.Hello`这个类时,会依次查找:

  - `<当前目录>\abc\xyz\Hello.class`
  - `C:\work\project1\bin\abc\xyz\Hello.class`
  - `C:\shared\abc\xyz\Hello.class`

- `classpath`的设定方法有两种:

  - 在系统环境变量中设置`classpath`环境变量，不推荐;

  - 在启动JVM时设置`classpath`变量，推荐:

    ```shell
    java -classpath .;C:\work\project1\bin;C:\shared abc.xyz.Hello
    
    java -cp .;C:\work\project1\bin;C:\shared abc.xyz.Hello
    ```

    没有设置系统环境变量，也没有传入`-cp`参数，那么JVM默认的`classpath`为`.`，即当前目录。
    ```java
    java abc.xyz.Hello
    ```

    在IDE中运行Java程序，IDE自动传入的`-cp`参数是当前工程的`bin`目录和引入的jar包。
    ```java
    C:\work> java -cp . com.example.Hello
    ```

    JVM根据classpath设置的`.`在当前目录下查找`com.example.Hello`，即实际搜索文件必须位于`com/example/Hello.class`。如果指定的`.class`文件不存在，或者目录结构和包名对不上，均会报错。

- **jar包**

  如果有很多`.class`文件，散落在各层目录中，肯定不便于管理。如果能把目录打一个包，变成一个文件，就方便多了。

  jar包就是用来干这个事的，它可以把`package`组织的目录层级，以及各个目录下的所有文件（包括`.class`文件和其他文件）都打成一个jar文件，这样一来，无论是备份，还是发给客户，就简单多了。

  jar包实际上就是一个zip格式的压缩文件，而jar包相当于目录。如果我们要执行一个jar包的`class`，就可以把jar包放到`classpath`中:
  ```java
  java -cp ./hello.jar abc.xyz.Hello
  ```

  这样JVM会自动在`hello.jar`文件里去搜索某个类。

  **如何创建jar包?**
  因为jar包就是zip包，所以，直接在资源管理器中，找到正确的目录，点击右键，在弹出的快捷菜单中选择“发送到”，“压缩(zipped)文件夹”，就制作了一个zip文件。然后，把后缀从`.zip`改为`.jar`，一个jar包就创建成功。

  ```shell
  package_sample
  └─ bin
     ├─ hong
     │  └─ Person.class
     │  ming
     │  └─ Person.class
     └─ mr
        └─ jun
           └─ Arrays.class
  ```

  这里需要特别注意的是, **jar包里的第一层目录,不能是`bin`**，而应该是`hong`、`ming`、`mr`。如果在Windows的资源管理器中看，应该长这样:
  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220815160800083.png" alt="image-20220815160800083" style="zoom:50%;" />
  如果长这样:
  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220815160851915.png" alt="image-20220815160851915" style="zoom:50%;" />

  **说明打包打得有问题，JVM仍然无法从jar包中查找正确的`class`，原因是`hong.Person`必须按`hong/Person.class`存放，而不是`bin/hong/Person.class`。**

  jar包还可以包含一个特殊的`/META-INF/MANIFEST.MF`文件，`MANIFEST.MF`是纯文本，可以指定`Main-Class`和其它信息。
  JVM会自动读取这个`MANIFEST.MF`文件，如果存在`Main-Class`，我们就不必在命令行指定启动的类名，而是用更方便的命令:

  ```java
  java -jar hello.jar
  ```

  java的开源工具: Maven;

##### class版本

Java 11对应的class文件版本是55, Java 17对应的class文件版本是61。
我们也可以用Java 17编译一个Java程序，指定输出的class版本要兼容Java 11（即class版本55），这样编译生成的class文件就可以在Java >=11的环境中运行。
两种方式:

- Javac 命令中用参数`--release`
  ```java
  javac --release 11 Main.java
  ```

  参数`--release 11`表示源码兼容Java 11，编译的class输出版本为Java 11兼容，即class版本55。

- 用参数`--source`指定源码版本，用参数`--target`指定输出class版本:
  ```java
  javac --source 9 --target 11 Main.java
  ```

  上述命令如果使用Java 17的JDK编译，它会把源码视为Java 9兼容版本，并输出class为Java 11兼容版本。

- 因此，如果运行时的JVM版本是Java 11，则编译时也最好使用Java 11，而不是用高版本的JDK编译输出低版本的class。

#### 模块

