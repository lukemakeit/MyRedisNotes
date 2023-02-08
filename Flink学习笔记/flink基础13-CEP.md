## Flink CEP

在大数据分析领域，一大类需求就是诸如PV、UV这样的统计指标，我们往往可以直接写SQL搞定;对于比较复杂的业务逻辑，SQL中可能没有对应功能的内置函数，那么我们也可以使用`DataStream API`,利用状态编程来进行实现。**不过在实际应用中，还有一类需求是要检测以特定顺序先后发生的一组事件， 进行统计或做报警提示，这就比较麻烦了。例如，网站做用户管理，可能需要检测“连续登录失败”事件的发生，这是个组合事件，其实就是“登录失败”和“登录失败”的组合;电商网站可能需要检测用户“下单支付”行为，这也是组合事件，“下单”事件之后一段时间内又会有“支付”事件到来，还包括了时间上的限制**

#### CEP是什么

**所谓CEP,其实就是“复杂事件处理(Complex Event Processing)”的缩写;而Flink CEP,就是Flink实现的一一个用于复杂事件处理的库(library)**

那到底什么是“复杂事件处理”呢? 就是可以在事件流里，检测到特定的事件组合并进行处理，比如说“连续登录失败”，或者“订单支付超时”等等。

具体的处理过程是，把事件流中的一个个简单事件，通过一定的规则匹配组合起来，这就是“复杂事件”;然后基于这些满足规则的一-组组复杂事件进行转换处理，得到想要的结果进行
输出。

总结起来，复杂事件处理(CEP)的流程可以分成三个步骤:

- 定义一个匹配规则
- 将匹配规则应用到事件流上，检测满足规则的复杂事件
-  对检测到的复杂事件进行处理，得到结果进行输出

#### 模式(pattern)

CEP的第一步所定义的匹配规则，我们可以把它叫作“模式”(Pattern)。模式的定义主要就是两部分内容:
-  每个简单事件的特征
- 简单事件之间的组合关系

当然，我们也可以进一步扩展模式的功能。比如:

匹配检测的时间限制;每个简单事件是否可以重复出现;对于事件可重复出现的模式，遇到一个匹配后是否跳过后面的匹配等等;

所谓“事件之间的组合关系”，一般就是定义“谁后面接着是谁”，也就是事件发生的顺序;我们把它叫作“近邻关系”。可以定义严格的近邻关系，也就是两个事件之前不能有任何其他事件;也可以定义宽松的近邻关系，即只要前后顺序正确即可，中间可以有其他事件。另外,还可以反向定义，也就是“谁后面不能跟着谁”。CEP做的事其实就是在流上进行模式匹配。根据模式的近邻关系条件不同，可以检测连续的事件或不连续但先后发生的事件;模式还可能有时间的限制,如果在设定时间范围内没有满足匹配条件，就会导致模式匹配超时( timeout)。

#### 应用场景

CEP主要用于实时流数据的分析处理。CEP可以帮助在复杂的、看似不相关的事件流中找出那些有意义的事件组合，进而可以接近实时地进行分析判断、输出通知信息或报警。这在企业项目的风控管理、用户画像和运维监控中，都有非常重要的应用。
- **风险控制**
设定一些行为模式，可以对用户的异常行为进行实时检测。当一个用户行为符合了异常行为模式，比如短时间内频繁登录并失败、大量下单却不支付(刷单)，就可以向用户发送通知信息，或是进行报警提示、由人工进一步判定用户是否有违规操作的嫌疑。这样就可以有效地控制用户个，人和平台的风险。

- **用户画像**

  利用CEP可以用预先定义好的规则，对用户的行为轨迹进行实时跟踪，从而检测出具有特定行为习惯的一些用户，做出相应的用户画像。基于用户画像可以进行精准营销，即对行为匹配预定义规则的用户实时发送相应的营销推广;这与目前很多企业所做的精准推荐原理是一样的。

- **运维监控**
  对于企业服务的运维管理，可以利用CEP灵活配置多指标、多依赖来实现更复杂的监控模式;

#### 快速上手

为了精简和避免依赖冲突，Flink会保持尽量少的核心依赖。所以核心依赖中并不包括任何的连接器(conncetor) 和库，这里的库就包括了SQL、CEP 以及ML等等。**所以如果想要在Flink集群中提交运行CEP作业，应该向Flink SQL那样将依赖的jar 包放在/ib目录下从这个角度来看，Flink CEP和Flink SQL一样，都是最项层的应用级API**

**一个简单实例:如果连续三次登录失败, 则输出报警信息。**

```java
package com.luke.flink.datastreamdemo;

import java.time.Duration;
import java.util.List;
import java.util.Map;

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.cep.CEP;
import org.apache.flink.cep.PatternSelectFunction;
import org.apache.flink.cep.PatternStream;
import org.apache.flink.cep.pattern.Pattern;
import org.apache.flink.cep.pattern.conditions.SimpleCondition;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class CepLoginFailDetectExample {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 1. 获取登录数据
        SingleOutputStreamOperator<CepLoginEvent> loginEventStream = env.fromElements(
                new CepLoginEvent("user 1", "192.168.0.1", "fail", 2000L),
                new CepLoginEvent("user 1", "192.168.0.2", "fail", 3000L),
                new CepLoginEvent("user 2", "192.168.1.29", "fail", 4000L),
                new CepLoginEvent("user 1", "171.56.23.10", "fail", 5000L),
                new CepLoginEvent("user 2", "192.168.1.29", "fail", 7000L),
                new CepLoginEvent("user 2", "192.168.1.29", "fail", 8000L),
                // 如果user 2最后一行数据timestamp是6000L, 那就不会输出user
                // 2连续三次登录失败的日志,因为时间确定了这条success日志会插队到前面
                new CepLoginEvent("user 2", "192.168.1.29", "success", 9000L))
                .assignTimestampsAndWatermarks(WatermarkStrategy.<CepLoginEvent>forBoundedOutOfOrderness(Duration.ZERO)
                        .withTimestampAssigner(new SerializableTimestampAssigner<CepLoginEvent>() {
                            @Override
                            public long extractTimestamp(CepLoginEvent element, long recordTimestamp) {
                                return element.timestamp;
                            }
                        }));

        // 2. 定义模式,连续三次登录失败
        Pattern<CepLoginEvent, CepLoginEvent> pattern = Pattern.<CepLoginEvent>begin("first") // 第一次登录失败事件
                .where(new SimpleCondition<CepLoginEvent>() {
                    @Override
                    public boolean filter(CepLoginEvent value) throws Exception {
                        return value.eventType.equals("fail");
                    }
                })
                .next("second")
                .where(new SimpleCondition<CepLoginEvent>() {
                    @Override
                    public boolean filter(CepLoginEvent value) throws Exception {
                        return value.eventType.equals("fail");
                    }
                })
                .next("third")
                .where(new SimpleCondition<CepLoginEvent>() {
                    @Override
                    public boolean filter(CepLoginEvent value) throws Exception {
                        return value.eventType.equals("fail");
                    }
                });
        // 3. 将模式应用到数据流上,检查复杂事件
        PatternStream<CepLoginEvent> patternStram = CEP.pattern(loginEventStream.keyBy(event -> event.userId), pattern);

        // 4. 将检测到的复杂事件提取出来,进行处理得到报警信息输出
        SingleOutputStreamOperator<String> warningStream = patternStram
                .select(new PatternSelectFunction<CepLoginEvent, String>() {
                    // key为啥string呢? 其实map中保存的就是事件名称,如上面的 first、second、third
                    // 为啥value是list呢，因为一个事件可能重复发生,list中就会保存多个事件。本示例中,list长度总是唯一
                    @Override
                    public String select(Map<String, List<CepLoginEvent>> map) throws Exception {
                        // 提取复杂事件中的三次登录失败事件
                        CepLoginEvent firstFailEvent = map.get("first").get(0);
                        CepLoginEvent secondFailEvent = map.get("second").get(0);
                        CepLoginEvent thirdFailEvent = map.get("third").get(0);
                        return firstFailEvent.userId + " 连续三次登录失败! 登录时间:" + firstFailEvent.timestamp + ", "
                                + secondFailEvent.timestamp + ", " + thirdFailEvent.timestamp;
                    }
                });
        // 打印输出
        warningStream.print();

        env.execute();
    }
}
// 输出:
user 1 连续三次登录失败! 登录时间:2000, 3000, 5000
user 2 连续三次登录失败! 登录时间:4000, 7000, 8000
```

#### 模式 API(Pattern API)

Pattern API可以让我们定义各种复杂的事件组合规则，用于从事流中提取复杂事件。

##### 个体模式

- **基本形式**

  如上面，每个登录失败事件的选取规则，就是一个个体模式。如:

  ```java
  <CepLoginEvent>begin("first") // 第一次登录失败事件
                  .where(new SimpleCondition<CepLoginEvent>() {
                      @Override
                      public boolean filter(CepLoginEvent value) throws Exception {
                          return value.eventType.equals("fail");
                      }
                  })
  ```

  或者后面的:

  ```java
                  .next("second")
                  .where(new SimpleCondition<CepLoginEvent>() {
                      @Override
                      public boolean filter(CepLoginEvent value) throws Exception {
                          return value.eventType.equals("fail");
                      }
                  })
  ```

  这些都是个体模式，个体模式一般都会匹配接收一个事件。

- **量词(Quantifiers)**

  个体模式后面可以跟一个“量词”，用来指定循环的次数。从这个角度分类，个体模式可以包括“单例(singleton) 模式”和“循环(looping) 模式”。默认情况下，个体模式是单例模式，匹配接收一个事件;当定义了量词之后，就变成了循环模式，可以匹配接收多个事件。

  在循环模式中，对同样特征的事件可以匹配多次。比如我们定义个体模式为“匹配形状为三角形的事件”，再让它循环多次，就变成了“匹配连续多个三角形的事件”。注意这里的“连续”，只要保证前后顺序即可，中间可以有其他事件，所以是“宽松近邻”关系。

  在Flink CEP中，可以使用不同的方法指定循环模式，主要有:

  - **`oneOrMore()`**

    匹配事件出现一次或多次，假设a是一个个体模式，`a.oneOrMore()`表示可以匹配1个或多个a的事件组合。我们有时会用`a+`简单表示;

  - **`.time(times)`**

    匹配事件发生特定次数，比如`a.times(3)`表示aaa;

  - **`.times(fromTimes,toTimes)`**

    指定匹配事件出现的次数范围，最小次数为`fromTimes`,最大次数为`toTimes`。

  - **`.greedy()`**

    只能用在循环模式后，使当前循环模式变得“贪心**(greedy)**，也就是总是尽可能多地去匹配。例如`a.times(2, 4).greedy()`,如果出现了连续4个a,那么会直接把aaaa检测出来进行处理，其中任意2个a是不算匹配事件的;

  - **`.optional()`**

    当前模式是可选的，也就是说可以满足这个匹配条件，也可以不满足。

  ```java
  //匹配事件出现4次
  pattern.times(4);
  
  //匹配事件出现4次，或者不出现
  pattern.times(4).optional();
  
  //匹配事件出现2, 3或者4次
  pattern.times(2, 4);
  
  //匹配事件出现2, 3或者4次，并且尽可能多地匹配
  pattern.times(2, 4).greedy();
  
  //匹配事件出现2，3, 4次，或者不出现
  pattern.times(2，4).optional();
  
  //匹配事件出现2，3, 4次，或者不出现;并且尽可能多地匹配
  pattern.times(2，4).optional().greedy();
  
  //匹配事件出现1次或多次
  pattern.oneOrMore();
  
  //匹配事件出现1次或多次，并且尽可能多地匹配
  pattern.oneOrMore().greedy();
  
  //匹配事件出现1次或多次，或者不出现
  pattern.oneOrMore().optional();
  
  //匹配事件出现1次或多次，或者不出现;并且尽可能多地匹配
  pattern.oneOrMore().optional().greedy();
  
  //匹配事件出现2次或多次
  pattern.timesOrMore(2);
  
  //匹配事件出现2次或多次，并且尽可能多地匹配
  pattern.timesOrMore(2).greedy();
  
  //匹配事件出现2次或多次，或者不出现
  pattern.timesOrMore(2).optional();
  ```

  **正是因为个体模式可以通过量词定义为循环模式，一个模式能够匹配到多个事件，所以之前代码中事件的检测接收才会用Map中的一个列表(List) 来保存**。而之前代码中没有定义量词，都是单例模式，所以只会匹配一个事件，每个List中也只有一个元素:

  ```java
  LoginEvent first = map.get("first").get(0);
  ```

- **条件(Conditions)**

  对于条件的定义，主要是通过调用Pattern对象的`.where()`方法来实现的，主要可以分为简单条件、迭代条件、复合条件、终止条件几种类型。此外，也可以调用Pattern对象的`.subtype()`方法来限定匹配事件的子类型。接下来我们就分别进行介绍。

  - 限定子类型

    调用`.subtype()`方法可以为当前模式增加子类型限制条件。例如:
    ```
    pattern.subtype (SubEvent.class);
    ```
    这里SubEvent是流中数据类型Event的子类型。这时，只有当事件是SubEvent类型时，才可以满足当前模式pattern的匹配条件。

  - **简单条件(Simple Conditions)**

    简单条件是最简单的匹配规则，只根据当前事件的特征来决定是否接受它。本质上就是一个filter操作;

    代码中我们为.where(方法传入一个`SimpleCondition`的实例作为参数。`SimpleCondition`是表示“简单条件”的抽象类，内部有一个.filter(方法，唯一的参数就是当前事件。所以它可以当作`FilterFunction`来使用。

  - **迭代条件(Iterative Conditions)**

    简单条件只能基于当前事件做判断，能够处理的逻辑比较有限。**在实际应用中，我们可能需要将当前事件跟之前的事件做对比，才能判断出要不要接受当前事件。这种需要依靠之前事件来做判断的条件，就叫作“迭代条件”( Iterative Condition)**

    在FlinkCEP中，提供了`IterativeCondition`抽象类。这其实是更加通用的条件表达，查看源码可以发现`.where()`方法本身要求的参数类型就是`IterativeCondition`; 而之前的`SimpleCondition`是它的一个子类。

    在`IterativeCondition中同样需要实现一个`filter()`方法，不过与`SimpleCondition`中不同的是,**这个方法有两个参数:除了当前事件之外，还有一个上下文Context。调用这个上下文的`.getEventsForPattern()`方法，传入一个模式名称，就可以拿到这个模式中已匹配到的所有数据了**

    ```java
    // 1. oneOrMore() 很可能导致 一直匹配下去，数量无限制；每来一个Event都会调用 filter()函数哦
    // 2. 如何限制不再匹配呢? 这里filter()函数做了两个限制如果不是以A开头的用户,则不匹配了，以前ctx中匹配到的元素会丢弃,直到下一次匹配开始。如果sum >100则不匹配了,以前ctx中匹配到的元素会丢弃，直到下一次匹配开始；
    middle.oneOrMore()
    .where(new IterativeCondition<Event>() (
    	@override
    	public boolean filter(Event value, Context<Event> ctx) throws Exception (
    		//事件中的user必须以A开头
    		if(!value.user.startsWith("A")) [
    			return false;
    		}
    		int sum = value . amount;
    		//获取当前模式之前已经匹配的事件，求所有事件amount之和
    		for(Event event:ctx.getEventsForPattern ("middle")) (
    			sum += event.amount ;
    		)
    		//在总数量小于100时，当前事件满足匹配规则，可以匹配成功
    		return sum <100;
    )) ;
    ```

    通过 迭代条件，我们可以实现一些更复杂的需求，比如可以要求 "只有大于之前数据的平均值，才接受当前事件"。

    同时迭代条件中上下文Context也可以获取时间相关信息，如当前事件时间戳 和 当前处理时间(processing time)。

  - **组合条件 conbining Conditions**

    如果一个个体模式有多个限定条件，又该怎么定义呢?

    最直接的想法是，可以在简单条件或者迭代条件的`.filter()`方法中，增加多个判断逻辑。可以通过`if-else`的条件分支分别定义多个条件,也可以直接在return返回时给一个多条件的逻辑组合(与、或、非)。不过这样会让代码变得臃肿，可读性降低。更好的方式是独立定义多个条件，然后在外部把它们连接起来，构成一个“组合条件”(Combining Condition)。

    **最简单的组合条件，就是`.where()`后面再接一个`.where()`。因为前面提到过，一个条件就像是一个filter操作，所以每次调用`.where()`方法都相当于做了一次过滤，连续多次调用就表示多重过滤，最终匹配的事件自然就会同时满足所有条件。这相当于就是多个条件的“逻辑与”(AND)**

    **而多个条件的逻辑或(OR)， 则可以通过`.where()`后加一个`.or()`来实现。这里的`.or()`方法与`.where()`一样， 传入一个`IterativeCondition`作为参数,定义一 个独立的条件;它和之前`.where()`定义的条件只要满足一个，当前事件就可以成功匹配**
    当然，子类型限定条件(subtype)也可以和其他条件结合起来，成为组合条件

  - **终止条件(Stop Conditions)**

    对循环模式而言，还可以指定一个终止条件(Stop Condition)，表示遇到某个特定事件时当前模式就不在继续循环匹配了。

    终止条件的定义是通过调用模式对象的`.until()`方法来实现的，同样传入一个`IterativeCondition`作为参数。需要注意的是，**终止条件只与`oneOrMore()`或者`oneOrMore().optional()`结合使用**。因为在这种循环模式下，我们不知道后面还有没有事件可以匹配，只好把之前匹配的事件作为状态缓存起来继续等待,这等待无穷无尽;如果一直等下去，缓存的状态越来越多，最终会耗尽内存。所以这种循环模式必须有个终点，当`.until()`指定 的条件满足时，循环终止，这样就可以清空状态释放内存了。


##### 组合模式

有了定义好的个体模式，就可以尝试按一定的顺序把它们连接起来，定义一个完整的复杂事件匹配规则了。这种将多个个体模式组合起来的完整模式，就叫作"组合模式"(Combining Pattemn)，为了跟个体模式区分有时也叫作"模式序列"(Pattern Sequence)。

一个组合模式有以下形式：

```java
Pattern<Event,?> pattern = Pattern
	.<Event>begin("start").where(...)
	.next("next").wheref(...)
	.followedBy("follow").where(...)
	...
```

可以看到，组合模式确实就是一个“模式序列”，是用诸如 begin、 next、 followedBy 等表示先后顺序的“连接词”将个体模式串连起来得到的。在这样的语法调用中，每个事件匹配的条件是什么、各个事件之间谁先谁后、近邻关系如何都定义得一目了然。每一个“连接词”方法调用之后，得到的都仍然是一个 Pattern 的对象；所以从 Java 对象的角度看，组合模式与个体模式是一样的，都是 Pattern。

- **初始模式(Initial Pattern)**

  所有的组合模式，都必须以一个"初始模式"开头；而初始模式必须通过调用`Pattern`的静态方法`.begin()`来创建。如下所示:

  ```java
  Pattern<Event,?> start = Pattern.<Event>begin("start");
  ```

  - 这里我们调用 Pattern 的`.begin()`方法创建了一个初始模式。传入的 String 类型的参数就是模式的名称；
  - 而 begin 方法需要传入一个类型参数，这就是模式要检测流中事件的基本类型， 这里我们定义为 Event;
  - 调用的结果返回一个 Pattern 的对象实例;
  - Pattern 有两个泛型参数，第一个就是检测事件的基本类型 Event，跟 begin 指定的类型一致；第二个则是当前模式里事件的子类型，由子类型限制条件指定。我们这里用类型通配符(?)代替，就可以从上下文直接推断了;

- **近邻条件(Contiguity Conditions)**

  在初始模式之后，我们就可以按照复杂事件的顺序追加模式，组合成模式序列了。模式之间的组合通过一些"连接词"方法实现，这些连接词指明了先后事件之间有着怎样的近邻关系，这就是所谓的"近邻条件"(contiguity Contitions)
  
  Flink CEP提供了三种近邻关系:
  
  - 严格近邻(Strict Contiguity)
  
    **匹配的事件严格地按顺序一个接一个出现，中间不会有任何其他事件。代码中对应的就是 Pattern 的.next0方法，名称上就能看出来，“下一个”自然就是紧挨着的**
  
  - **宽松近邻(Relaxed Contiguity)**
  
    宽松近邻只关心事件发生的顺序，而放宽了对匹配事件的“距离”要求， 也就是说两个匹配的事件之间可以有其他不匹配的事件出现。代码中对应`.followedBy()`方法， 很明显这表示 “跟在后面”就可以，不需要紧紧相邻。
  
    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230103144118382.png" alt="image-20230103144118382" style="zoom: 50%;" />
  
  - **非确定性宽松近邻(Non-Deterministic Relaxed Contiguity)**
  
    这种近邻关系更加宽松。所谓“非确定性”是指<font style="color:red">可以重复使用之前己经匹配过的事件</font>；这种近邻条件下匹配到的不同复杂事件，可以以同一个事件作为开始，所以匹配结果一般会比宽松近邻更多。代码中对应`.followedByAny()`方法。
  
    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20230103144538210.png" alt="image-20230103144538210" style="zoom:50%;" />
  
  - **其他限定条件**
  
    除了上面提到的`next()`、`followedBy()`、`followedByAny()`可以分别表示三种近邻条件，我们还可以使用否定的"连接词"来组合个体模式。主要包括:
  
    - **`.notNext()`**
  
      表示一个模式匹配到的事件后面，不能紧跟某种事件。
  
    - **`.notFollowedBy()`**
  
      **表示前一个模式匹配到的事件后面，不会出现某种事件。这里需要注意，由于`noFollowedBy()`是没有严格限定的；流数据不停地到来，我们永远不能保证之后 “不会出现某种事件”。所以一个模式序列不能以 `notFollowedBy()`结尾，这个限定条件主要用来表示“两个事件中间不会出现某种事件”**
  
    另外，Flink CEP 中还可以为模式指定一个时间限制，这是通过调用`.within()`方法实现的。**方法传入一个时间参数，这是模式序列中第一个事件到最后一个事件之间的最大时间间隔，只有在这期间成功匹配的复杂事件才是有效的**。一个模式序列中只能有一个时间限制，调用.within0的位置不限；如果多次调用则会以最小的那个时间间隔为准。
  
    ```java
    // 严格近邻条件
    Pattern<Event,?> strict = start.next("middle").where (...）；
    
    //宽松近邻条件
    Pattern<Event,?> relaxed = start.followedBy("middle").where (...);
    
    // 非确定性宽松近邻条件
    Pattern<Event,?> nonDetermin = start.followedByAny("middle").where(. . .）；
    
    //不能严格近邻条件
    Pattern<Event, ?> strictNot = start.notNext("not").where(...);
    
    // 不能宽松近邻条件
    Pattern<Event,?＞ relaxedNot = start.notFollowedBy("not").where(...);
    
    // 时间限制条件
    middle.within (Time.seconds(10))；
    ```
  
  - **循环模式中的近邻条件**
  
    循环模式是指上面用两次定义多次匹配的情况，如:
  
    ```java
    //匹配事件出现4次
    pattern.times(4);
    
    //匹配事件出现4次，或者不出现
    pattern.times(4).optional();
    
    //匹配事件出现2, 3或者4次
    pattern.times(2, 4);
    
    //匹配事件出现1次或多次
    pattern.oneOrMore();
    ...
    ```
  
    - **`.time(3).consecutive()`: 表示三次紧跟的匹配关系，相当于调用了三次`.next()`**
    - `.time(3).allowCombinations()`: 相当于调用了三次`.followedByAny()`
  
  ##### 模式组
  
  一般来说，代码中定义的模式序列，就是我们在业务逻辑中匹配复杂事件的规则。不过在有些非常复杂的场景中，可能需要划分多个“阶段°，每个“阶段”又有一连串的匹配规则。为了应对这样的需求，Flink CEP 允许我们以“嵌套”的方式来定义模式。
  
  直接以一个模式序列作为参数，就将模式序列又一次连接组合起来了。这样得到的就是一个“模式组” (Groups of Patterns )。
  
  **在模式组中，每一个模式序列就被当作了某一阶段的匹配条件，返回的类型是一个GroupPattern。而 GroupPatern 本身是 Pattern 的子类：所以个体模式和组合模式能调用的方法， 比如`times()`、`oneOrMore()`、`optional()`之类的量词，模式组一般也是可以用的**
  
  具体在代码中的应用如下所示：
  
  ```java
  // 以模式序列作为初始模式
  Pattern<Event,?>startPattern.begin(Pattern.<Event>begin("start start").where(...))
  .followedBy("start middle").where(...)
  
  //在start 后定义严格近邻的模式序列，并重复匹配两次
  Pattern<Event, ?＞strict = start. next (
  	Pattern.<Event>begin("next start").where(...)
  	.followedBy("next middle").where(...)
  ).times(2);
  
  //在start 后定义宽松近邻的模式序列,并重复匹配一次或多次
  Pattern<Event， ?> relaxed = start. followedBy(
  	Pattern. <Event>begin("followedby start") .where (..)
  	.followedBy("followedby middle").where (...)
  ).oneOrMore();
  ```

##### 匹配后跳过策略

