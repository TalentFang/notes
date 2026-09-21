
```
# HTTP 代理
export http_proxy=http://10.15.136.20:8080

# HTTPS 代理
export https_proxy=http://10.15.136.20:8080
export no_proxy=127.0.0.1,localhost,10.23.18.156,113.122.0.0/16,113.123.0.0/16,fc00::/64,fd00::/108
```

只编译redisearch模块：
redis-8.6.6/modules/Makefile 第一行只保留redisearch

编译命令：
```
# 编译
export CC=gcc
export CXX=g++
export CFLAGS="-Wno-error"
export CXXFLAGS="-Wno-error"

# kylin
make CMAKE_ARGS="-DCMAKE_C_COMPILER=/usr/local/gcc-11.4.0/bin/gcc -DCMAKE_CXX_COMPILER=/usr/local/gcc-11.4.0/bin/g++" MALLOC=libc CC=gcc CXX=g++ LDFLAGS="-lssl -lcrypto -lz" CFLAGS="-Wno-error" CXXFLAGS="-Wno-error" BUILD_TLS=yes BUILD_WITH_MODULES=yes IGNORE_MISSING_DEPS=1 -j$(nproc) all

# euler
make MALLOC=libc CC=gcc CXX=g++ LDFLAGS="-lssl -lcrypto -lz" CFLAGS="-Wno-error" CXXFLAGS="-Wno-error" BUILD_TLS=yes BUILD_WITH_MODULES=yes IGNORE_MISSING_DEPS=1 -j$(nproc) all

# 安装
make PREFIX=/data/comm/redis-8.6.6 install
```


**gcc版本低于gcc 8.x**
```
# 这个方法比较耗时（约1-2小时），但最可靠
cd /tmp
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/gcc/gcc-11.4.0/gcc-11.4.0.tar.gz
tar -xzf gcc-11.4.0.tar.gz
cd gcc-11.4.0
./contrib/download_prerequisites
./configure --prefix=/usr/local/gcc-11.4.0 --enable-languages=c,c++ --disable-multilib
make -j$(nproc)
make install

# 设置环境变量
export PATH=/usr/local/gcc-11.4.0/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/gcc-11.4.0/lib64:$LD_LIBRARY_PATH

# 永久生效
echo 'export PATH=/usr/local/gcc-11.4.0/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/gcc-11.4.0/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc

# 验证
gcc --version
```

**拷贝**
glibc/glibc++ 拷贝到目标机器
ln -s 创建软连接
```
cd /root/
scp root@192.168.112.78:/usr/local/gcc-11.4.0/lib64/libstdc++.so.6.0.29  .
scp root@192.168.112.78:/usr/local/gcc-11.4.0/lib64/libgcc_s.so.1  .
ln -sf libstdc++.so.6.0.29 libstdc++.so.6
```

export LD_LIBRARY_PATH 路径


**openssl-devel,zlib-devel** 
```
# openssl-deve,zlib-devell** 
yum install -y openssl-devel zlib-devel

# rustc
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env


# cmake
yum install -y cmake

# clang 
yum install -y clang clang-devel
```
PARAM_KAFKA_AUTH_EXTERNAL=SCRAM-SHA-512


**rustc ：** 
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

**cmake:**
```
yum install -y cmake
```
**clang**
```
yum install -y clang clang-devel
```




修改Makefile
```

vi /root/redis-8.6.6/src/Makefile
```
**找到 `FINAL_LIBS` 并添加 `-lz`**：


```
# 加载模块

```

