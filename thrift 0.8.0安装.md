# 一、常用小工具安装
该步骤非必须，可跳过

安装brew（Mac OS的软件包管理器）：https://brew.sh/index_zh-cn

使用brew安装下列工具包

brew install autoconf 

brew install automake

brew install libtool

# 二、Install boost
下载： http://sourceforge.net/projects/boost/files/boost/1.60.0/boost_1_60_0.tar.gz/download

安装过程耗时较长，约25-30分钟

主要命令：

解压：tar -zxvf boost_1_60_0.tar.gz

移动至固定目录：mv ~/Downloads/boost_1_60_0.tar ~/Programs/boost_1_60_0.tar

编译：./bootstrap.sh

执行配置：sudo ./b2 threading=multi address-model=64 variant=release stage install

​

# 三、Install libevent
brew install libevent

# 四、Install thrift
编译： ./configure --prefix=/usr/local/ --with-boost=/usr/local --with-libevent=/usr/local --without-csharp --without-erlang --without-go --without-haskell --without-ruby --without-cpp --without-perl --without-php --without-php_extension --without-python

安装： sudo make install

验证查看是否安装成功： thrift -version 


