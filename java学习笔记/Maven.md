## Maven

**是什么?**

Maven本质是一个项目管理工具，将项目开发和管理过程抽象成一个项目对象模型(POM)。
POM(Project Object Model): 项目对象模型

![image-20220821225351338](/Users/lukexwang/Library/Application Support/typora-user-images/image-20220821225351338.png)

**Maven作用**

- 项目构建: 提供标准的、跨平台的自动化项目构建方式
- 依赖管理: 方便快捷的管理项目依赖的资源(jar包)，避免资源间版本冲突问题;
- 统一开发结构: 提供标准的、统一的项目结构

  <img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20220821225639970.png" alt="image-20220821225639970" style="zoom:50%;" />

#### Maven 基础概念

**仓库:用于存储资源,包含各种jar包**

- 中央仓库

- 本地仓库, 是中央仓库的一个子集

- 私服: 公司的仓库

  具有版权的资源，包含购买或自主研发的jar包

  缓存，加快jar包拉去速度

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220821231104539.png" alt="image-20220821231104539" style="zoom:50%;" />

**坐标**

- **官方仓库:[https://repo1.maven.org/maven2/](https://repo1.maven.org/maven2/)**
- [Https://mvnrepository.com](Https://mvnrepository.com)
- Maven坐标主要组成
  - groupId: 定义当前Maven项目隶属组织名称(通常是域名反写, 如 org.mybatis)
  - artifactId: 定义房钱Maven项目名称(通常是模块名称，如CRM、SMS)
  - version: 定义当前项目版本号
  - Packaging: 定义项目的打包方式
- Maven坐标的作用: 使用唯一标识，唯一性定位资源位置，通过该标识可以将资源的识别与下载工作交给机器完成。

#### Maven: 仓库配置

**本地仓库配置**

默认本地仓库(`localRepository`)位置:`$HOME/.m2/`
文件: `/usr/local/apache-maven-3.8.6/conf/settings.xml`

```xml
  <!-- localRepository
   | The path to the local repository maven will use to store artifacts.
   |
   | Default: ${user.home}/.m2/repository
  <localRepository>/path/to/local/repo</localRepository>
  -->
```

设置自己的本地仓库位置:

```shell
$ mkdir /data/home/lukexwang/Document/maven_repo
$ vim /usr/local/apache-maven-3.8.6/conf/settings.xml
<localRepository>/data/home/lukexwang/Document/maven_repo</localRepository>
```

**镜像仓库设置**
镜像仓库配置:

```xml
// 文件: /usr/local/apache-maven-3.8.6/conf/settings.xml
<!-- 配置具体的仓库下载镜像 -->
<mirror>
        <!-- 此镜像的唯一标识符,用来区分不同的mirror元素 -->
        <id>aliyunmaven</id>
        <!-- 对哪种仓库进行镜像,简单来说就是替换哪个仓库 -->
        <mirrorOf>central</mirrorOf>
        <!-- 镜像名称 -->
        <name>阿里云公共仓库</name>
        <!-- 镜像URL -->
        <url>https://maven.aliyun.com/repository/central</url>
</mirror>
```

**全局setting和用户setting的区别**

- 刚刚设置的是全局settting;

- 每个用户在自己的`localRepository`中可以设置属于自己的`settings.xml`，`cp /usr/local/apache-maven-3.8.6/conf/settings.xml /data/home/lukexwang/Document/maven_repo`;

  然后更改`/data/home/lukexwang/Document/maven_repo/settings.xml`

#### 制作maven项目

**Maven项目结构: 项目名java-project**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220823072336502.png" alt="image-20220823072336502" style="zoom:50%;" />

- `main`: 写主要程序
  - java 源程序
  - resources: 配置文件
- `test`:写测试程序的;
  - java 源程序
  - resources: 配置文件

**mvn创建工程**

```shell
$ mvn archetype:generate 
-DgroupId={project-packaging} 
-DartifactId={project-name} 
-DarchetypeArtifactId=maven-archetype-quickstart # 使用的模版名称
-DinteractiveMode=false
```

**创建java工程:**

```shell
mvn archetype:generate  -DgroupId=com.luke  -DartifactId=java-project  -DarchetypeArtifactId=maven-archetype-quickstart -Dversion=0.0.1-snapshot -DinteractiveMode=false
```

**创建web工程**

```shell
mvn archetype:generate  -DgroupId=com.luke  -DartifactId=web-project  -DarchetypeArtifactId=maven-archetype-webapp -Dversion=0.0.1-snapshot -DinteractiveMode=false
```

如果执行这个命令一直卡住，如下。则先删除`~/.m2`

![image-20220823070435217](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220823070435217.png)

**Maven项目构建命令**

```java
mvn compile # 编译
mvn clean #清理编译的target
mvn test #测试,执行测试用例,同时测试的详细日志和结果保存到 target/surefire-reports 中
mvn package #打包,先compile,再test,最后在target下面会生成jar包
mvn install #安装到本地仓库 <localRepository>/path/to</localRepository>
```

#### 依赖管理

- 依赖是指当前项目运行所需的jar,一个项目可以设置多个依赖

-  格式:

  ```xml
    <dependencies>
      <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>3.8.1</version>
        <scope>test</scope>
      </dependency>
  <dependency>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
      <version>1.2.17</version>
  </dependency>
  <!-- 下面是依赖着本地其他项目 -->
  <dependency>
      <groupId>com.itheima</groupId>
      <artifactId>project03</artifactId>
      <version>1.0-SNAPSHOT</version>
  </dependency>
    </dependencies>
  ```

**依赖传递**

- 直接依赖: 在当前项目中通过依赖配置建立的依赖关系
- 间接依赖: 被资源的资源如果依赖其他资源，当前项目间接依赖其他资源
- 依赖传递冲突问题
  - **路径优先: 当依赖中出现相同的资源，层级越深，优先级越低，层级越浅，优先级越高**
  - **声明优先: 当资源在相同层级被依赖时，配置顺序靠前的覆盖顺序靠后的**
  - **特殊优先: 当同级配置了相同资源的不同版本，后配置的覆盖先配置的**

**可选依赖**

可选依赖指**对外隐藏当前所依赖的资源——不透明。别人看不到我用了啥**

```xml
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
    <optional>true</optional>
</dependency>
```

**排除依赖**

**排除依赖是指主动断开依赖的资源，被排除的资源无需指定版本。比如`project03`中用了`log4j 1.2.10`版本，现在想排除。则可以这样**

```xml
    <dependency>
      <groupId>com.luke</groupId>
      <artifactId>project03</artifactId>
      <version>1.0-SNAPSHOT</version>
      <exclusions>
        <exclusion>
          <groupId>log4j</groupId>
          <artifactId>log4j</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
```

**依赖范围**

- 依赖的jar默认情况可以再任何地方使用，可以通过`scope`标签设定其作用范围
- 作用范围
  - 主程序范围有效(main文件夹范围内)
  - 测试程序范围有效(test文件夹范围内)
  - 是否参与打包(package指令范围内)

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220824234412146.png" alt="image-20220824234412146" style="zoom:50%;" />

**依赖范围的传递性**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220824234922374.png" alt="image-20220824234922374" style="zoom:50%;" />

#### 声明周期和插件

**项目构建生命周期**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220913234548586.png" alt="image-20220913234548586" style="zoom:50%;" />

```java
compile => text => package(打包) =>install
```

- Maven对项目构建生命周期划分为三套

  - clean: 清理工作

    - pre-clean: 执行一些需要在clean之前完成的工作
    - clean: 移除所有上一次构建生成的文件
    - post-clean: 执行一些需要在clean之后立刻完成的工作  

  - default: 核心工作,例如编译，测试，打包，部署等

    ![image-20220913232737814](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220913232737814.png)

    执行test，则把前面的`validate`、`initialize`...`compile`全部执行。

  - site: 产生报告,发布站点等 

**插件**

- 官网: [https://maven.apache.org/plugins/index.html](https://maven.apache.org/plugins/index.html)

- 插件与生命周期内的阶段绑定，在执行到对应声明周期时指定对应的插件功能;
- 默认maven在各个声明周期上绑定有预设的功能
- 通过插件可以自定义其他功能

```xml
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-source-plugin</artifactId>
			<version>2.2.1</version>
			<executions>
				<execution>
					<goals>
						<goal>jar</goal>
					</goals>
					<phase>generate-test-resources</phase>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```

- 最重要的是**`<goals>`、`<phase>`**
- **<goals>: 的值有`jar`、`test-jar`等,`jar`:正式代码打包, `test-jar`:测试代码打包**
- **`<phase>`: 代表在哪个阶段执行打包，上面的是在`generate-test-resources`阶段执行打包**

