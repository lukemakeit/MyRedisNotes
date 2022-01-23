### GLOG 使用

### 安装

- 安装`gflags`: [https://github.com/gflags/gflags/releases](https://github.com/gflags/gflags/releases)

  ```shell
  cd gflags-2.2.2
  mkdir /usr/local/gflags-2.2.2
  
  # 共享库
  cmake -DCMAKE_INSTALL_PREFIX=/usr/local/gflags-2.2.2 -DBUILD_SHARED_LIBS=on ..
  make && make install
  
  # 静态库
  cmake -DCMAKE_INSTALL_PREFIX=/usr/local/gflags-2.2.2 ..
  make && make install
  ```

- glog

  下载地址: [https://github.com/google/glog/releases](https://github.com/google/glog/releases)

  编译链接方法方法一:

  ```cmake
  find_path(GLOG_HEADER glog
  	HINTS src/thirdparty/glog/src
  )
  message("GLOG_HEADER = ${GLOG_HEADER}")
  SET(BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
  add_subdirectory(src/thirdparty/glog glog EXCLUDE_FROM_ALL)
  
  file(GLOB SOURCE "src/capture/*.cpp")
  
  add_executable(myRedisCapture ${SOURCE})
  set_target_properties(myRedisCapture PROPERTIES LINK_FLAGS "-static-libgcc -static-libstdc++")
  target_include_directories(myRedisCapture PUBLIC ${GLOG_HEADER})
  target_link_libraries(myRedisCapture glog)
  ```

  编译链接方法方法二:

  ```cmake
  #编译
  cd src/thirdparty/glog
  cmake -DBUILD_SHARED_LIBS=off  -S . -B build -G "Unix Makefiles"
  cd build && make
  
  #项目中链接
  set(GLOG_ROOT_DIR "${PROJECT_SOURCE_DIR}/src/thirdparty/glog")
  include_directories("${GLOG_ROOT_DIR}/src")
  include_directories("${GLOG_ROOT_DIR}/build")
  add_library(glog_static STATIC IMPORTED)
  set_target_properties(glog_static 
                          PROPERTIES IMPORTED_LOCATION
                          "${GLOG_ROOT_DIR}/build/libglog.a")
  add_executable(myRedisCapture src/capture/main.cpp)
  target_link_libraries(myRedisCapture PRIVATE glog_static)
  ```

  