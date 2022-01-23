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

   