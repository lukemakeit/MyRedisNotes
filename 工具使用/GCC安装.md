## GCC 7.5.0安装

1. 用国内镜像地址下载:

   ```sh
   wget https://mirrors.ustc.edu.cn/gnu/gcc/gcc-7.5.0/gcc-7.5.0.tar.gz
   tar -zxf gcc-7.5.0.tar.gz
   cd gcc-7.5.0
   ```

2. 下载依赖:

   ```sh
   wget https://mirrors.ustc.edu.cn/gnu/gmp/gmp-6.1.0.tar.bz2
   wget https://mirrors.ustc.edu.cn/gnu/mpc/mpc-1.0.3.tar.gz
   wget https://mirrors.ustc.edu.cn/gnu/mpfr/mpfr-3.1.4.tar.bz2
   wget http://ftp.ntua.gr/mirror/gnu/gcc/infrastructure/isl-0.16.1.tar.bz2 //这个可能比较慢
   ```

3. 安装`gmp`、`mpc`、`mpfr`、`isl`

   ```shell
   tar -xjf gmp-6.1.0.tar.bz2
   cd gmp-6.1.0/
   mkdir /usr/local/gmp-6.1.0/
   ./configure --prefix=/usr/local/gmp-6.1.0/
   make
   make install
   
   tar -xjf mpfr-3.1.4.tar.bz2
   mkdir /usr/local/mpfr-3.1.4
   cd mpfr-3.1.4/
   ./configure --prefix=/usr/local/mpfr-3.1.4/ --with-gmp=/usr/local/gmp-6.1.0/
   make
   make install
   
   tar -zxf mpc-1.0.3.tar.gz
   cd mpc-1.0.3/
   mkdir /usr/local/mpc-1.0.3/
   ./configure --prefix=/usr/local/mpc-1.0.3/ --with-gmp=/usr/local/gmp-6.1.0/ --with-mpfr=/usr/local/mpfr-3.1.4/
   make
   make install
   
   tar -xjf isl-0.16.1.tar.bz2
   cd isl-0.16.1
   mkdir /usr/local/isl-0.16.1
   ./configure --prefix=/usr/local/isl-0.16.1/ --with-gmp-prefix=/usr/local/gmp-6.1.0/ --with-mpfr=/usr/local/mpfr-3.1.4/ --with-mpc=/usr/local/mpc-1.0.3/
   make
   make install
   ```

4. 安装gcc-7.5.0

   ```shell
   cd gcc-7.5.0 && mkdir gcc-Builder-7.5.0 && cd gcc-Builder-7.5.0
   export LD_LIBRARY_PATH=/usr/local/isl-0.16.1/lib:/usr/local/gmp-6.1.0/lib:/usr/local/mpfr-3.1.4/lib:/usr/local/mpc-1.0.3/lib:$LD_LIBRARY_PATH
   
   ../configure --prefix=/usr/local/gcc-7.5.0/ --enable-checking=release --enable-languages=c,c++ --disable-multilib --with-gmp=/usr/local/gmp-6.1.0/ --with-mpfr=/usr/local/mpfr-3.1.4/ --with-mpc=/usr/local/mpc-1.0.3/ --with-isl=/usr/local/isl-0.16.1/
   
   nohup make -j 24 >> gcc.log 2>&1 & //可能需要执行很久
   make install
   ```


### `gcc`常见选项含义

文章: [gcc 编译 选项 汇总](https://zhuanlan.zhihu.com/p/347611674)

### `g++`场景选项含义

- `-g`: 打开debug开关;
- `-Wall`： 打开大部分warning提示；
- `-O` 或`-O2`开启编译器优化;
- `-o` 输出文件的名字;
- `-c`输出一个object文件(`.o`)
- `-l`指定一个include文件夹;
- `-L`指定一个寻找库文件的目录;
- `-l` 链接一个动态库;

### `gcc` 和 `g++`的区别

原文: [g++以及gcc的区别](https://www.cnblogs.com/samewang/p/4774180.html)
首先说明：gcc 和 GCC 是两个不同的东西。
GCC:GNU Compiler Collection(GUN 编译器集合)，它可以编译C、C++、JAV、Fortran、Pascal、Object-C、Ada等语言。
gcc是GCC中的GUN C Compiler（C 编译器）
g++是GCC中的GUN C++ Compiler（C++编译器）
一个有趣的事实就是，就本质而言，gcc和g++并不是编译器，也不是编译器的集合，它们只是一种驱动器，根据参数中要编译的文件的类型，调用对应的GUN编译器而已，比如，用gcc编译一个c文件的话，会有以下几个步骤：

- Step1：Call a preprocessor, like cpp;
- Step2：Call an actual compiler, like cc or cc1.
- Step3：Call an assembler, like as.
- Step4：Call a linker, like ld

更准确的说法是：gcc调用了C compiler，而g++调用了C++ compiler。
gcc和g++的主要区别

1. 对于 `.c`和`.cpp`文件，gcc分别当做`c`和`cpp`文件编译(c和cpp的语法强度是不一样的);

2. 对于` .c`和`.cpp`文件，`g++`则统一当做cpp文件编译;

3. 使用`g++`编译文件时，**<mark style="color:red">`g++`会自动链接标准库STL，而gcc不会自动链接STL</mark>**;

4. gcc在编译C文件时，可使用的预定义宏是比较少的;

5. gcc在编译cpp文件时/g++在编译c文件和cpp文件时（这时候gcc和g++调用的都是cpp文件的编译器），会加入一些额外的宏，这些宏如下:

   ```cpp
   #define __GXX_WEAK__ 1
   #define __cplusplus 1
   #define __DEPRECATED 1
   #define __GNUG__ 4
   #define __EXCEPTIONS 1
   #define __private_extern__ extern
   ```

6. 在用gcc编译c++文件时，为了能够使用STL，需要加参数`–lstdc++` ，但这并不代表 `gcc –lstdc++` 和 `g++`等价



