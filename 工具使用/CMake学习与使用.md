## CMake 学习与使用

#### 安装

- `open-ssl`安装

  下载地址: [https://github.com/openssl/openssl/tags](https://github.com/openssl/openssl/tags)

  ```shell
  tar -zxf openssl-openssl-3.0.1.tar.gz
  cd openssl-openssl-3.0.1
  mkdir /usr/local/openssl-3.0.1/
  ./Configure --prefix=/usr/local/openssl-3.0.1
  make
  make install
  export OPENSSL_ROOT_DIR=/usr/local/openssl-3.0.1
  ```

- 源码下载: [https://cmake.org/download/](https://cmake.org/download/)

  ```shell
  cd cmake-3.22.1
  mkdir /usr/local/cmake
  ./bootstrap --prefix=/usr/local/cmake
  make -j 6
  make install
  ```

  添加命令`export PATH=/usr/local/cmake/bin:$PATH`

### 最常用的命令与示例

- `cmake_mininum_required`

  定义cmake的最低兼容版本，如`CMAKE_MINIMUM_REQUIRED(VERSION 2.5 FATAL_ERROR)`。
  如果版本低于2.5,则出现严重错误，整个过程终止。

- `project`

  项目名称

  ```cmake
  # cmake 最低版本要求
  cmake_mininum_required(VERSION 3.13)
  #项目名称
  project(cmake_study)
  #设置
  set(CMAKE_CXX_STANDARD 11)
  #编译源码生成目标
  add_executable(cmake_study src/main.cc)
  ```

- `set`

  - **给一般变量赋值,语法:`set(<variable> <value>... [PARENT_SCOPE])`**

  **每一个新的目录或者函数都会创建新的作用域**。
  
  当在新的作用域创建一般变量且后面加上PARENT_SCOPE，该变量可以在父目录或者调用新函数的函数上起作用。
  
  示例:`set(src_list main.c other.c)` 后续就能用`${src_list}`代替`main.c other.c`
  
  ```cmake
  set(FOO "x") //FOO作用域为当前作用域
  set(FOO "x" PARENT_SCOPE) //FOO作用域跳上一级
  ```
  
  - **给缓存变量赋值(cache variables)**
  
    缓存变量可以理解为当第一次运行cmake时，这些变量缓存到一份文件中(即编译目录下的CMakeCache.txt)。
  
    当再次运行cmake时，这些变量会直接使用缓存值，可以利用ccmake或者cmake-gui等工具来重新赋值。
  
    缓存变量在整个cmake运行过程中都可以起作用。
  
    语法: `set(<variable> <value>... CACHE <type> <docstring> [FORCE])`
  
    - variable: 变量名
    - value:  可以是0个，1个或多个，当value值为空时，方法同unset，用于取消设置的值;
    - CACHE: 关键字，说明是缓存变量设置;
    - type类型，必须为下列中的一种:
      - BOOL: 有ON/OFF, 两种值;
      - FILEPATH: 文件的全路径
      - PATH: 目录路径
      - STRING: 字符串
      - INTERENAL: 字符串，将变量变为内部变量，即`cmake-gui`不会向用户显式这类变量。
    - Docstring: 帮助性文字(字符串)
    - `[FORCE]`: 当使用CACHE时，且缓存(cache)中没有该变量时，变量被创建并存入缓存(cache)中，如果原缓存(cache)中有该变量，也不会改变原缓存中该变量的值，除非后面使用FORCE。
  
    示例:
  
    ```cmake
    #原缓存中没有FOO则将FOO赋值为x且存入cache中;
    #原缓存中有FOO则不做任何改变，即便原cache中FOO存的不是x
    set(FOO, "x" CACHE STRING) 
    
    #即便原cache中存在FOO也会创建另一个FOO
    set(FOO, "x" CACHE STRING "" FORCE)
    ```
  
    <mark style="color:red">**一个工程利用`ADD_SUBDIRECTORY()`添加子工程，子工程有自己的CMakeList.txt。此时即可利用`SET( CACHE <type> <docstring> [FORCE])`来完成对子工程的传值。**</mark>
  
    参考问题: [Overriding a default option(...) value in CMake from a parent CMakeLists.txt](https://stackoverflow.com/questions/3766740/overriding-a-default-option-value-in-cmake-from-a-parent-cmakelists-txt)
  
    ```cmake
    SET(FOO_BUILD_SHARED OFF CACHE BOOL "Build libfoo shared library")
    add_subdirectory(extern/foo)
    ```
  
    我自己在使用[redis-plus-plus](https://github.com/sewenew/redis-plus-plus#installation)时也遇到类似的问题，设置hiredis的路径:
  
    ```cmake
    SET(CMAKE_PREFIX_PATH /usr/local/hiredis CACHE BOOL "" FORCE)
    SET(REDIS_PLUS_PLUS_BUILD_TEST OFF CACHE BOOL "" FORCE)
    add_subdirectory(src/thirdparty/redis-plus-plus redis++ EXCLUDE_FROM_ALL)
    ```
  
    理论上来说下面方法也是可以的，但是我试过很多次就是不行，不知道为啥:
  
    ```cmake
    add_subdirectory(src/thirdparty/redis-plus-plus redis++ EXCLUDE_FROM_ALL)
    target_compile_definitions( redis++
    							PRIVATE  
    							-DCMAKE_PREFIX_PATH=/usr/local/hiredis 
    							-DREDIS_PLUS_PLUS_BUILD_TEST=OFF
    						)
    ```
  
- `message`

  打印日志

  如: `message("CMAKE_BINARY_DIR=>${CMAKE_BINARY_DIR}")`
  
- `FILE`

  文件操作指令。如

  `file(GLOB SOURCE "src/*.cpp")`，将`src/main.c src/other.c`存入到变量`SOURCE`中。
  
- `AUX_SOURCE_DIRECTORY`

  示例: `aux_source_directory(./src hello_src)`

  作用: 把当前路径下src目录下的所有原文件件放到`hello_src`变量中
  
  注意:
  
  - `aux_source_directory`不会递归包含子目录，仅包含指定的dir目录;
  
  - 我们也可以写成:`aux_source_directory(${CMAKE_CURRENT_LIST_DIR}/src ${hello_src})`
  
- `add_definitions`

  为源文件添加由`-D`引入的宏定义;

  命令格式是:`add_definitions(-DFOO -DBAR ...)`

  例如: `add_definitions(-DWIN32)`

  ![image-20220110235250871](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220110235250871.png)

- `target_compile_definitions`

  为目标设置编译标志。

  ```cmake
  target_compile_definitions(aa PRIVATE EX3)
  ```

  上面的代码在编译目标`aa`时候，会为编译器添加`-DEX3`标识。

  如果目标是`aa`是一个library,同时scope是`PUBLIC` 或 `INTERFACE`而不是`PRIVATE`，那么所有连接`aa`的目标都会包含定义`-DEX3`;

  更多:[target_compile_options()](https://cmake.org/cmake/help/v3.0/command/target_compile_options.html)

-  `add_lbrary`

  `add_library()`**用于从源文件中创建一个静态/共享库**。如:

  ```cmake
  add_library(hello_library STATIC src/Hello.cpp)
  ```

  该命令可以用于从`add_library`函数调用的 源文件中创建一个名为`libhello_library.a`的静态库(static library)。

  下面命令将创建一个名为`libhello_library.so`的共享库:

  ```cmake
  add_library(hello_library SHARED src/Hello.cpp)
  ```

  `add_library`还可以用来为library创建别名(`alias target`)。别名目标 只是为了 在读取上下文中可以代替真实目标名称。

  ```cpp
  add_library(hello_library SHARED src/Hello.cpp)
  
  add_library(hello::library ALIAS hello_library) #创建别名
  
  add_executable(hello_binary src/main.cpp )
  
  target_link_libraries( hello_binary PRIVATE hello::library)
  ```

  <mark style="color:red">**导入一个已经存在的库，一般配合`set_target_properties`使用。`set_target_properties`用于指定导入库的路径:**</mark>

  ```cmake
  add_library(test STATIC IMPORTED)
  set_target_properties(  test #指定目标库名称
                          PROPERTIES IMPORTED_LOCATION #指明要设置的参数
                          ${CMAKE_CURRENT_LIST_DIR}/lib/libtest.a #设定导入库的路径)
  ```

  ```cmake
  #导入静态库
  add_library(jemalloc STATIC IMPORTED)
  set_target_properties(  jemalloc 
                          PROPERTIES IMPORTED_LOCATION
                          "${JEMALLOC_ROOT_DIR}/lib/libjemalloc_pic.a")
  #导入共享库
  add_library(jemalloc_SHARED SHARED IMPORTED)
  set_target_properties(  jemalloc_SHARED 
                          PROPERTIES IMPORTED_LOCATION 
                          "${JEMALLOC_ROOT_DIR}/lib/libjemalloc.so")
                          
  add_executable(main main.cpp)
  target_link_libraries(main jemalloc)
  
  还可以写成这样:
  add_executable(main main.cpp)
  target_link_libraries(main ${JEMALLOC_ROOT_DIR}/lib/libjemalloc_pic.a)
  ```

- `include_directories`

  如:`include_directories(./include)`

  作用: 当前目录(也就是`CMakeLists.txt`所在目录)下的include文件夹加入到`include`路径中。

  一般会写成:`include_directories(${CMAKE_LIST_DIR}/include)`

  语法: `INCLUDE_DIRECTORIES([AFTER|BEFORE] [SYSTEM] dir1 [dir2 ...])`

  作用:

  - 把`dir1`, `[dir2 …]`这（些）个路径添加到当前CMakeLists及其子CMakeLists的头文件包含路径中;

  - AFTER 或者 BEFORE 指定了要添加的路径是添加到原有包含列表之前或之后;

  - 若指定 SYSTEM 参数，则把被包含的路径当做系统包含路径来处理;

- `target_include_directories`

  当编译某个目标需要`include`不同的目录时，能通过`target_include_directories()`让编译器知道。

  当编译该目标工程时，编译器会添加`-I`标识(如`-I/directory/path`)

  如:

  ```cmake
  target_include_directories(target PRIVATE  ${PROJECT_SOURCE_DIR}/include
  ```

- `link_directories`

  如:`link_directories(${CMAKE_CURRENT_LIST_DIR}/lib)`

  指定可以去`${CMAKE_CURRENT_LIST_DIR}/lib`**这个目录下寻找 静态库/共享库**。

- `target_link_libraries`

  当目标需要和某个 静态库/共享库 链接才能完成时。我们可以通过`target_link_libraries`来完成该行为。

  ```cmake
  target_link_libraries(hello_binary PRIVATE  hello_library)
  ```

- `set_property` 不是很懂，干嘛的?

  每个域的对象都能设置若干属性，比如有如下几个域: `GLOBAL`	、`DIRECTORY`、`TARGET`

  语法:

  ```cpp
  set_property(<GLOBAL | 
              DIRECTORY [dir] | 
              TARGET [target ...] | 
              SOURCE [src1 ...] | 
              TEST [test1 ...] | 
              CACHE [entry1 ...]>
               [APPEND] 
               PROPERTY <name> [value ...])
  ```

  比如命令:

  ```cpp
  add_executable(a ...)
  set_property(
      TARGET a
      APPEND PROPERTY
          INCLUDE_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}"
  )
  上面的set_property等价于:
  target_include_directories(a PRIVATE  ${CMAKE_CURRENT_SOURCE_DIR})
  ```

- `set_target_properties`

- **`CMAKE_BUILD_TYPE`**

  Cmake有很多内置的build配置，可用于编译项目。这些配置指定了优先级别以及调试信息是否包含在最终二进制文件中。

  - `Release` - 编译时候添加`-03 -DNDEBUG` 标志(flags);
  - `Debug` - 编译时添加`-g`标志(flags);
  - `MinSizeRel` - 编译时添加`-0s -DNDEBUG`标志(flags);
  - `RelWithDebInfo` - 编译时添加`-O2 -g -DNDEBUG`标志(flags);

  比如现在我们发现cmake编译出的二进制无法调试，我们就可以通过下面命令用Debug模式编译。

  ```cmake
  set(CMAKE_BUILD_TYPE Debug)
  ```

  参考问题:[Creating symbol table for gdb using cmake](https://stackoverflow.com/questions/7990844/creating-symbol-table-for-gdb-using-cmake)

- **设置默认C++ 编译标志**

  默认`CMAKE_CXX_FLAGS`为空 或者根据`CMAKE_BUILD_TYPE`选择合适的标志。

  为了设置额外默认的编译标志，你能在top level `CMakeLists.txt`中添加下面的命令:

  ```cmake
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DEX2" CACHE STRING "Set C++ Compiler Flags" FORCE)
  ```

  注意: 上面命令中`CACHE STRING "Set C++ Compiler Flags" FORCE`会强制在`CMakeCache.txt`设置上该变量。

  类似上面设置`CMAKE_CXX_FLAGS`，我们还可以设置更多选项:

  - 通过`CMAKE_C_FLAGS`设置C编译器标志;

  - 通过`CMAKE_LINKER_FLAGS`设置连接标志;

  一旦`CMAKE_CXX_FLAGS`和`CMAKE_C_FLAGS`设置后，所有目录 子目录中的目标都会都会带上这两个标志编译。
  
- `add_subdirectory`

  原文文档: [Cmake命令之add_subdirectory介绍](https://www.jianshu.com/p/07acea4e86a3)

  命令语法: 

  ```cmake
  add_subdirectory (source_dir [binary_dir] [EXCLUDE_FROM_ALL])
  添加一个子目录并构建该子目录。
  ```

  - `source_dir` :**必选参数**,该参数指定一个子目录，子目录下应该包含`CMakeLists.txt`文件和源代码文件。子目录可以是相对路径也可是是绝对路径，如果是相对路径，则是相对当前目录的一个相对路径；
  - `binary_dir`:**可选参数**, 该参数指定一个目录，用于存放输出文件。也可以是相对路径，也可是是绝对路径，如果没指定则默认的输出目录是`source_dir`;
  - `EXCLUDE_FROM_ALL`:**可选参数**。指定该参数，则子目录下的目标不会被父目录下的目标文件包含进去，父目录的`CMakeLists.txt`不会构建子目录的目标文件，必须在子目录下显示构建。例外情况: **当父目录的目标依赖于子目录的目标，则子目录的目标仍然会被构建出来以满足依赖关系(例如使用了target_link_libraries)**

  举例说明:

  ```cmake
  ├── CMakeLists.txt    #父目录的CMakeList.txt
  ├── main.cpp    #源文件，包含main函数
  ├── sub    #子目录
   └── CMakeLists.txt    #子目录的CMakeLists.txt
   └── test.h    #子目录头文件
   └── test.cpp    #子目录源文件
  ```

  ```cpp
  //  sub/test.cpp  
  #include "test.h"
  #include <iostream>
  
  void test(std::string str)
  {
      std::cout << str << std::endl;
  }
  ```

  ```cmake
  //  sub/test.h
  #include <string>
  
  void test(std::string str);
  ```

  ```cmake
  # sub/CMakeLists.txt
  cmake_minimum_required(VERSION 3.10.2)
  project(sub)
  add_library(sub test.cpp)
  ```

  - **场景1: 父目录`CMakeLists.txt`的`add_subdirectory`只指定了`source_dir`**

    ```cmake
    # 父目录下的CMakeLists.txt
    cmake_minimum_required(VERSION 3.10.2)
    project(test)
    
    add_subdirectory(sub) 
    ```

    在父目录下调用`cmake .`构建之后，在sub目录下会出现`libsub.a`库，说明当不指定`binary_dir`，输出目标文件就会放到`source_dir`目录下。

  - **场景2: 父目录`CMakeLists.txt`的`add_subdirectory`指定了`source_dir`和`binary_dir`**

    ```cmake
    # 父目录下的CMakeLists.txt
    cmake_minimum_required(VERSION 3.10.2)
    project(test)
    
    add_subdirectory(sub output) 
    ```

    在父目录下调用`cmake .`构建之后，在`output`目录下会出现`libsub.a`库，`sub`目录下则没有`libsub.a`。说明当指定`binary_dir`，输出目标文件就会放到`binary_dir`目录下。

  - **场景3:父目录`CMakeLists.txt`的`add_subdirectory` 指定了`EXCLUDE_FROM_ALL`选项**

    ```cmake
    # 父目录下的CMakeLists.txt
    cmake_minimum_required(VERSION 3.10.2)
    project(test)
    
    add_subdirectory(sub output EXCLUDE_FROM_ALL) 
    add_executable(test main.cpp)
    ```

    在父目录下调用`cmake .`构建之后，在`output`目录或`sub`目录下`不会`出现`libsub.a`库，说明当指定`EXCLUDE_FROM_ALL`选项，子目录的目标文件不会生成。

  - **场景4: 父目录`CMakeLists.txt`的`add_subdirectory` 指定了`EXCLUDE_FROM_ALL`选项，且父目录的目标文件依赖子目录的目标文件**

    ```cmake
    # 父目录下的CMakeLists.txt
    cmake_minimum_required(VERSION 3.10.2)
    project(test)
    
    add_subdirectory(sub output EXCLUDE_FROM_ALL) 
    add_executable(test main.cpp)
    target_link_libraries(test sub)
    ```

    在父目录下调用`cmake .`构建之后，在`output`目录会出现`libsub.a`库，说明即使指定`EXCLUDE_FROM_ALL`选项，当父目录目标文件对子目录目标文件存在依赖关系时，子目录的目标文件仍然会生成以满足依赖关系。

- `find_path` 

  简短模式: `find_path(<VAR> name1 [path1 path2 ...])`

  搜索包含某个文件的路径，结果保存在`<VAR>`中，**结果默认会创建Cache条目**；如果没有找到`<VAR>-NOTFOUND`就会被设置，在下次以相同的变量名再次调用`find_path()`时，该命令会再次尝试搜索该文件。

  ```cpp
  FIND_PATH(
    LIBDB_CXX_INCLUDE_DIR
    db_cxx.h 
    /usr/include/ 
    /usr/local/include/ 
  )
  ```

  示例:

  ![image-20220112011716559](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220112011716559.png)

  ```cmake
  find_path(HIREDIS_HEADER 
    hiredis
   	HINTS src/thirdparty
  )
  target_include_directories(myapp PUBLIC ${HIREDIS_HEADER})
  后续myapp的函数中就可以使用 include "hiredis/hiredis.h"
  ```

  额外的搜索路径还包括:

  1. 在新版本 3.12中，`find_package(<PackageName>)`的默认搜索路径包括 本地变量`<PackageName>_ROOT`和环境变量`<PackageName>_ROOT`代表的路径。

     还包括`${CMAKE_LIBRARY_ARCHITECTURE}/include/<arch>`

  2. `find_path`的搜索路径:

     `${CMAKE_LIBRARY_ARCHITECTURE}/include/<arch>`

     `${CMAKE_PREFIX_PATH}/include`

     `${CMAKE_INCLUDE_PATH}`

     `${CMAKE_FRAMEWORK_PATH}`

- `find_library`

  简短模式: `find_library(<VAR> names [path1 path2 ...] DOC REQUIRED)`

  用于在指定路径下查找库。如果找到则将结果保存在`<VAR>`中，同时为结果创建Cache条目。如果没有找到`<VAR>-NOTFOUND`就会被设置，在下次以相同的变量名再次调用`find_library()`时，该命令会再次尝试搜索该库。

  `NAMES`指定需要搜索的一个或多个库名称;

  `PATHS`除了默认位置外，还需要搜索的目录。

  `DOC`保存一些帮助信息;

  `REQUIRED`如果未找到任何内容，则停止处理并显示错误消息;

  ```cmake
  find_library(CURL_LIBRARY
      NAMES curl curllib libcurl_imp curllib_static
      HINTS "${CMAKE_PREFIX_PATH}/curl/lib"
  )
  ```

  默认的搜索路径还包括

  `${CMAKE_LIBRARY_ARCHITECTURE}/lib/<arch>`

  `${CMAKE_PREFIX_PATH}/lib`

  `${CMAKE_INCLUDE_PATH}`

  `${CMAKE_FRAMEWORK_PATH}`

- **add_custom_command**

  原文链接: [cmake：添加自定义操作](https://zhuanlan.zhihu.com/p/95771200)

  该指令用于添加自定义命令，实现某些操作。比如，编译之前进行一些文件拷贝操作等。

  两种使用方式:

  - **方式一: 配合 `add_custom_target` 使用，该命令生成 `add_custom_target` 的依赖**

    ```cmake
    add_custom_command(OUTPUT output1 [output2 ...]
                        COMMAND command1 [ARGS] [args1...]
                        [COMMAND command2 [ARGS] [args2...] ...]
                        [MAIN_DEPENDENCY depend]
                        [DEPENDS [depends...]]
                        [BYPRODUCTS [files...]]
                        [IMPLICIT_DEPENDS <lang1> depend1
                                         [<lang2> depend2] ...]
                        [WORKING_DIRECTORY dir]
                        [COMMENT comment]
                        [DEPFILE depfile]
                        [JOB_POOL job_pool]
                        [VERBATIM] [APPEND] [USES_TERMINAL]
                        [COMMAND_EXPAND_LISTS])
    ```

    - **`OUTPUT`: 目标文件名称，代表下面的COMMAND**
    - **`COMMAND`: 需执行的命令**
    - **`DEPENDS`: 执行命令时需要的依赖**

    示例:

    ```cmake
    cmake_minimum_required(VERSION 3.5)
    project(test)
    add_executable(${PROJECT_NAME} main.c)
    add_custom_command(OUTPUT printout 
                       COMMAND ${CMAKE_COMMAND} -E echo compile finish
                       VERBATIM
                      )
    add_custom_target(finish
                      DEPENDS printout
                      )
    ```

    `add_custom_command`生成一个名叫printout的"文件"(并非真是生成),其代表下方的 `COMMAND ${CMAKE_COMMAND} -E echo compile finish` 命令;

    `add_custom_target`"生成"目标`finish`，其依赖上方的`printout`。

    通过`--target finish`就会去生成`target finish`，进而执行依赖`printout`代表的命令。

    <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220111203641759.png" alt="image-20220111203641759" style="zoom:67%;" />

    更加实际的一个示例:

    在编译[Tendis](https://github.com/Tencent/Tendis/blob/dev-2.2/CMakeLists.txt)时，需要先确保`jemalloc`编译完成，`jemalloc`只支持通过`make`编译，所以不能通过`add_subdirectory()`来完成。

    正确的做法:

    ```cmake
    #tendisplus/CMakeLists.txt
    #添加自定义目标,用于实际执行jemalloc编译命令
    add_custom_command(OUTPUT 
    "${JEMALLOC_ROOT_DIR}/lib/libjemalloc.a"
    "${JEMALLOC_ROOT_DIR}/lib/libjemalloc.so"
                    COMMAND ${CMAKE_MAKE_PROGRAM} #其实就是make命令
                    DEPENDS "${JEMALLOC_ROOT_DIR}/include/jemalloc/jemalloc.h"
                    WORKING_DIRECTORY ${JEMALLOC_ROOT_DIR}
                    COMMENT "Making external library jemalloc")
    add_custom_target(build_jemalloc
                    DEPENDS "${JEMALLOC_ROOT_DIR}/lib/libjemalloc.a")
    
    #手动设置 jemalloc的 共享/静态库
    add_library(jemalloc_SHARED SHARED IMPORTED)
    add_library(jemalloc STATIC IMPORTED)
    set_target_properties(jemalloc PROPERTIES IMPORTED_LOCATION "${JEMALLOC_ROOT_DIR}/lib/libjemalloc_pic.a")
    set_target_properties(jemalloc_SHARED PROPERTIES IMPORTED_LOCATION "${JEMALLOC_ROOT_DIR}/lib/libjemalloc.so")
    
    #添加 jemalloc的 共享/静态库 对 target:build_jemalloc的依赖
    add_dependencies(jemalloc build_jemalloc)
    add_dependencies(jemalloc_SHARED build_jemalloc)
    
    #将 jemalloc 添加到 SYS_LIBS 中
    list(APPEND SYS_LIBS jemalloc)
    
    #tendisplus/src/tendisplus/server/CMakeLists.txt
    #编译tendisplus server时，连接SYS_LIBS
    add_executable(tendisplus main.cpp)
    target_link_libraries(tendisplus server_params server ${SYS_LIBS})
    ```

  - **方式二: 单独使用。在生成目标文件(使用`add_executable()`或`add_library()`生成的文件)时自动执行`add_custom_command`指定的命令**

    ```cmake
    add_custom_command(TARGET <target>
                        PRE_BUILD | PRE_LINK | POST_BUILD
                        COMMAND command1 [ARGS] [args1...]
                        [COMMAND command2 [ARGS] [args2...] ...]
                        [BYPRODUCTS [files...]]
                        [WORKING_DIRECTORY dir]
                        [COMMENT comment]
                        [VERBATIM] [USES_TERMINAL]
                        [COMMAND_EXPAND_LISTS])
    ```

    `TARGET`：由 `add_executable()` 或 `add_library()` 生成的目标文件名称；
    `PRE_BUILD|PRE_LINK|POST_BUILD`:分别表示编译之前执行命令,链接之前执行命令,生成目标文件后执行命令；

    ```cmake
    cmake_minimum_required(VERSION 3.5)
    project(test)
    add_executable(${PROJECT_NAME} main.c)
    add_custom_command(TARGET ${PROJECT_NAME} #生成目标文件之前/之后 执行某个命令
                       POST_BUILD # 生成目标文件后执行命令
                       COMMAND ${CMAKE_COMMAND} -E echo compile finish
                       VERBATIM
                      )
    ```

    ![image-20220111205717325](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220111205717325.png)

- **add_custom_target**

  cmake本身支持两种目标文件:可执行程序(由`add_executable`生成）和库文件(由`add_library`生成)。
  使用`add_custom_target`可添加自定义目标文件，用于执行某些指令。

  ```cmake
  add_custom_target(Name [ALL] [command1 [args1...]]
                     [COMMAND command2 [args2...] ...]
                     [DEPENDS depend depend depend ... ]
                     [BYPRODUCTS [files...]]
                     [WORKING_DIRECTORY dir]
                     [COMMENT comment]
                     [JOB_POOL job_pool]
                     [VERBATIM] [USES_TERMINAL]
                     [COMMAND_EXPAND_LISTS]
                     [SOURCES src1 [src2...]])
  ```

  - `NAME`: 目标名称;
  - `ALL`: 我们默认使用`cmake .. && make`并不会执行`add_custom_target`中的内容，需要使用`cmake --target <Name>`才会执行。但是如果指定了参数`ALL`,那么`cmake .. && make`也会执行`add_custom_target`中的内容；
  - `COMMAND`：需要执行的命令;
  - `DEPENDS`:执行命令时需要的依赖;
  - `COMMENT`：在构建时执行命令之前显示给定消息;
  - `WORKING_DIRECTORY`：使用给定的当前工作目录执行命令。如果它是相对路径，它将相对于对应于当前源目录的构建树目录;

  **add_custom_target**的功能看起来很像`Makefile`中的结构:

  ```makefile
  target ... : prerequisites ...  
      command
  ```

  例如:

  ```makefile
  clean:
          -rm ./$(SRV_NAME)
  build:clean
          CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -o $(SRV_NAME) -v ./cmd/tendisk8s
  ```

  下面是`add_custom_target`的示例:

  ```cmake
  cmake_minimum_required(VERSION 3.0)
  project(test)
  
  #自定义的一个target
  add_custom_target(myTask
    COMMAND ${CMAKE_COMMAND} -E echo myTask donee
  )
  
  #finish依赖myTask先完成
  add_custom_target(finish 
    DEPENDS myTask
    COMMAND ${CMAKE_COMMAND} -E echo compile finish 
  )
  ```

  ![image-20220111211644853](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220111211644853.png)

- `execute_process`

  ```cmake
  execute_process(COMMAND <cmd1> [args1...]]
                  [COMMAND <cmd2> [args2...] [...]]
                  [WORKING_DIRECTORY <directory>]
                  [TIMEOUT <seconds>]
                  [RESULT_VARIABLE <variable>]
                  [OUTPUT_VARIABLE <variable>]
                  [ERROR_VARIABLE <variable>]
                  [INPUT_FILE <file>]
                  [OUTPUT_FILE <file>]
                  [ERROR_FILE <file>]
                  [OUTPUT_QUIET]
                  [ERROR_QUIET]
                  [OUTPUT_STRIP_TRAILING_WHITESPACE]
                  [ERROR_STRIP_TRAILING_WHITESPACE])
  ```

  执行命令，将结果保存到cmake变量 or 文件中。比如你想通过git命令读取版本号，在代码中使用。

  注意: 每个进程的标准输出流向 下一个进程的标准输入。所有的流程使用一个单独的stderr错误管道。

  - `COMMAND`: 需要执行的命令，启动子进程来执行。注意`>`(输出重定向)这样的脚本操作符也会被当做普通参数处理。

    同时使用`INPUT_`、`OUTPUT_`、`	ERROR_`来重定向`stdin`、`stdout`、`stderr`。

  - `WORKING_DIRECTORY`:  工作目录。如果它是相对路径，它将相对于对应于当前源目录的构建树目录;

  - `TIMEOUT`: 超时时间，单位秒;

  - `RESULT_VARIABLE`: 结果变量，变量被设置为包含最后一个子进程的返回码(return code) 或当错误发生时，描述错误状态的字符串。

  - `OUTPUT_VARIABLE`、`ERROR_VARIABLE`:  `OUTPUT_VARIABLE`保存的是stdout的输出内容，`ERROR_VARIABLE`保存的是stderr的输出内容。如果为2个管道命名了相同的名字，他们的输出将按照产生顺序被合并。

  - `INPUT_FILE`,` OUTPUT_FILE`, `ERROR_FILE`: 输入文件，输出文件，错误文件。命名的文件将分别与第一个子进程的stdin，最后一个子进程的stdout，所有进程的stderr相关联。

  - `OUTPUT_QUIET`,`ERROR_QUIET`: 标准输出和标准错误的结果将被静默地忽略。

  示例:

  ```cpp
  execute_process(COMMAND touch aaa.jpg
                  COMMAND touch bbb.png
                  COMMAND ls
                  COMMAND grep -E "png|jpg"
                  OUTPUT_FILE pics)
  
  execute_process(
          COMMAND sh -c "echo ${ARG_TO_PASS}"
          OUTPUT_VARIABLE _RES
          ERROR_QUIET
  )
  ```

  关于`execute_process`传参的注意事项，可参考问题: [cmake execute_process COMMAND](https://stackoverflow.com/questions/43684051/cmake-execute-process-command)

  错误示范:

  ```cmake
  #不能这么传参，直接把$MAKE_CMD中的完整内容当成一个可执行文件了
  set(MAKE_CMD "${CMAKE_CURRENT_SOURCE_DIR}/makeHeaders.sh ${CMAKE_CURRENT_SOURCE_DIR} ${INC_DIR}")
  MESSAGE("COMMAND: ${MAKE_CMD}")
  execute_process(COMMAND "${MAKE_CMD}"   
     RESULT_VARIABLE CMD_ERROR
     OUTPUT_FILE CMD_OUTPUT)
  ```

  正确示例:

  ```cmake
  set(MAKE_CMD "${CMAKE_CURRENT_SOURCE_DIR}/makeHeaders.sh")
  MESSAGE("COMMAND: ${MAKE_CMD} ${CMAKE_CURRENT_SOURCE_DIR} ${INC_DIR}")
  execute_process(COMMAND ${MAKE_CMD} ${CMAKE_CURRENT_SOURCE_DIR} ${INC_DIR}
     RESULT_VARIABLE CMD_ERROR
     OUTPUT_FILE CMD_OUTPUT
  )
  
  set(MAKE_CMD "${CMAKE_CURRENT_SOURCE_DIR}/makeHeaders.sh")
  list(APPEND MAKE_CMD ${CMAKE_CURRENT_SOURCE_DIR})
  list(APPEND MAKE_CMD ${INC_DIR})
  execute_process(COMMAND ${MAKE_CMD}
     RESULT_VARIABLE CMD_ERROR
     OUTPUT_FILE CMD_OUTPUT
  )
  ```

### 常见问题

1. 如何设置使用C++ 11 or C++17?

   参考问题: [How do I activate C++ 11 in CMake?](https://stackoverflow.com/questions/10851247/how-do-i-activate-c-11-in-cmake)、[Setting CMAKE_CXX_STANDARD to various values](https://stackoverflow.com/questions/48026483/setting-cmake-cxx-standard-to-various-values)

   对于CMake≥3.1,`set(CMAKE_CXX_STANDARD 17)`是最佳方法。

   为了兼容老版本，可以这么写:

   ```cmake
   macro(use_cxx11)
     if (CMAKE_VERSION VERSION_LESS "3.1")
       if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
         set (CMAKE_CXX_FLAGS"${CMAKE_CXX_FLAGS} -std=gnu++11")
       endif ()
     else ()
       set (CMAKE_CXX_STANDARD 11)
     endif ()
   endmacro(use_cxx11)
   
   use_cxx11()
   ```

   我们未了让用户能自定义，可以这么写:

   ```cpp
   option(Barry_CXX_STANDARD "C++ standard" 11)
   set(CMAKE_CXX_STANDARD Barry_CXX_STANDARD)
   
   用户就可以通过 cmake -DBarry_CXX_STANDARD=17 来使用 c++17版本了。
   ```

2. 确定是哪种编译器？是gcc 还是 其他的？

   ```cmake
   if(CMAKE_COMPILER_IS_GNUCC)
   ```

   还可以用`CMAKE_<LANG>_COMPILER_ID`，参考文档:[CMAKE_<LANG>_COMPILER_ID](https://cmake.org/cmake/help/latest/variable/CMAKE_LANG_COMPILER_ID.html#variable:CMAKE_%3CLANG%3E_COMPILER_ID)

3. cmake已经为了定义了很多变量，查询地址: [cmake Useful Variables](https://gitlab.kitware.com/cmake/community/-/wikis/doc/cmake/Useful-Variables)

   | 变量名                   | 大致含义                                                     |
   | ------------------------ | ------------------------------------------------------------ |
   | PROJECT_SOURCE_DIR       | 项目的根目录，即包含 PROJECT() 命令的CMakeLists.txt 最近的目录 |
   | CMAKE_SOURCE_DIR         | 包含最顶层CMakeLists.txt的目录                               |
   | CMAKE_CURRENT_SOURCE_DIR | 当前正在处理的CMakeLists.txt所在的目录                       |
   | CMAKE_VERSION            | cmake的版本号                                                |
   | CMAKE_COMMAND            | 就是cmake命令，比如在add_custom_command中执行cmake命令<br />就需要用${CMAKE_COMMAND} |
   |                          |                                                              |

4. C++中如何使用`threads`?

   参考问题: [cmake and libpthread](https://stackoverflow.com/questions/1620918/cmake-and-libpthread)

   CMake 3.1.0+

   ```cmake
   set(THREADS_PREFER_PTHREAD_FLAG ON)
   find_package(Threads REQUIRED)
   target_link_libraries(my_app PRIVATE Threads::Threads)
   ```

   CMake 2.8.12+

   ```cpp
   find_package(Threads REQUIRED)
   if(THREADS_HAVE_PTHREAD_ARG)
     set_property(TARGET my_app PROPERTY COMPILE_OPTIONS "-pthread")
     set_property(TARGET my_app PROPERTY INTERFACE_COMPILE_OPTIONS "-pthread")
   endif()
   if(CMAKE_THREAD_LIBS_INIT)
     target_link_libraries(my_app "${CMAKE_THREAD_LIBS_INIT}")
   endif()
   ```

   
