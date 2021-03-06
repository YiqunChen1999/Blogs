---
title: 如何编译 caffe 源码
tags: Programming
articles:
  # data_source: site.sample_page
  type: brief
  shou_readmore: false
  show_info: true
  show_excerpt: false
article_header:
  type: cover
  image:
    src: /Images/Mathematics/IMG_1027.PNG
---
<!--more-->

# 如何在 Ubuntu 18.04LTS 上安装 caffe

**⚠ 警告：编译 caffe 十分麻烦，依赖报错无穷无尽，所以文章比较长，看来又有人要接受 caffe 的毒打了**

**由于我尚未成功解决跟显卡驱动相关的问题，因此暂时没有和 CUDA 相关的详细步骤，如果有成功编译 GPU 版本的小伙伴，欢迎交流**

## 检查依赖

依赖部分主要靠阅读官方源码编译部分的[依赖部分文档](https://caffe.berkeleyvision.org/installation.html)以及编译所需的 Makefile.config, 以下为检查并安装依赖的过程。

### 安装特定版本的 gcc & g++

`sudo apt install build-essential`

`sudo apt install gcc-6 g++-6`

修改 gcc 和 g++ 的软连接：

`sudo rm /usr/bin/gcc`

`sudo rm /usr/bin/g++`

`sudo ln -s /usr/bin/gcc-6 /usr/bin/gcc`

`sudo ln -s /usr/bin/g++-6 /usr/bib/g++`

### CUDA(optional)

想要运行 GPU 版本的 caffe 需要安装CUDA, 推荐的 CUDA 版本为 CUDA7+, 当然 CUDA6.x 也可以。首先检查 CUDA 版本：

`nvcc --version` 或者 `cat /usr/local/cuda/version.txt`

注意：`nvidia-smi` 中报告的 CUDA 版本指的是 driver version, 而 `nvcc --version` 指的是 runtime version, driver version 一般高于或者等于 runtime version. 

如果没有装 CUDA, 可以按照[官方文档](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#runfile-installation)安装。

### cuDNN(optional)

cuDNN 是一个加速软件包，可以按照[官方文档](https://docs.nvidia.com/deeplearning/cudnn/install-guide/index.html)安装。

查看 cuDNN 的安装位置：

`whereis cudnn`

根据 cuDNN 的位置查看版本：


`cat path_to_cudnn_header_file | grep CUDNN_MAJOR -A 2`

### ATLAS

安装 ATLAS, 可以直接搜索如何安装，也可以参考以下命令：

`sudo apt-get install libatlas-base-dev liblapack-dev libblas-dev`

### OpenBlas

我在最后一步 `make runtest` 的时候因为没有安装这个包报过错，因此这里我也把这个包当作依赖装上：

`sudo apt-get install libopenblas-dev`

### BOOST>=1.55

同样可以搜索如何安装，也可以先用 `apt search boost` 查找相应的软件包，以下命令可供参考：

`sudo apt-get install libboost-all-dev`

### Google Protocol Buffers

注意，该依赖应该安装 2.5 或者 2.6 版本，否则会因为版本问题出错。

从官方 Github 下载指定版本的 protocol: 

`wget https://github.com/protocolbuffers/protobuf/releases/download/v2.6.1/protobuf-2.6.1.zip`

或者打开[下载页面](https://github.com/protocolbuffers/protobuf/releases/tag/v2.6.1)下载自己喜欢的压缩文件。

根据官方[安装文档](https://github.com/protocolbuffers/protobuf/blob/master/src/README.md)编译源码。

解压：

`unzip protobuf-2.6.1.zip`

进入文件夹：

`cd protobuf-2.6.1.zip`

编译源码：

`./autogen.sh`

如果报错：`./autogen.sh: 38: ./autogen.sh: autoreconf: not found`

则安装即可：

`sudo apt-get install autoconf autogen`

继续编译：

`./configure`

`make`

`make check`

`sudo make install`

`sudo ldconfig # refresh shared library cache.`

注意：检查自己系统是否安装了 3.x 版本的 protoc, 如果有，请先删除/暂时关闭：

`whereis protoc`

`path_to_protoc --version`

### 安装 glog

`sudo apt-get install -y libgoogle-glog-dev`

### 安装 gflags

`sudo apt-get install libgflags-dev`

### 安装 hdf5

`sudo apt-get install libhdf5-serial-dev`

创建软链接：

`sudo ln -s /usr/lib/x86_64-linux-gnu/libhdf5_serial.so.100 /usr/lib/x86_64-linux-gnu/libhdf5.so`

`sudo ln -s /usr/lib/x86_64-linux-gnu/libhdf5_serial_hl.so.100 /usr/lib/x86_64-linux-gnu/libhdf5_hl.so`

### 安装 opencv3.x

`sudo apt-get install libopencv-dev`

### 安装 lmdb

`sudo apt install lmdb-utils`

`sudo apt-get install libgflags-dev libgoogle-glog-dev liblmdb-dev`

### 安装 leveldb

`sudo apt-get install libleveldb-dev`

### 安装 libsnappy

`sudo apt-get install libsnappy-dev`

## 修改编译文件 Makefile

编译过程中可能会产生如下错误：

`.build_release/lib/libcaffe.so: undefined reference to google::protobuf::`

参考 GitHub 中的[解决方案](https://github.com/BVLC/caffe/issues/3046), 为了避免未定义对 protobuf 引用的错误，首先运行：

`pkg-config --cflags --libs protobuf`

然后复制其输出内容，在 Makefile 中大约 428 行处找到：

`LINKFLAGS += -pthread -fPIC $(COMMON_FLAGS) $(WARNINGS)`

在其后面添加一行：

`LINKFLAGS += -pthread -I/usr/local/include -L/usr/local/lib -lprotobuf -pthread -lpthread`

在约 184 行处找到

`LIBRARIES += glog gflags protobuf boost_system boost_filesystem m`

并在其后添加

`hdf5_serial_hl hdf5_serial`

## 修改配置文件

复制配置文件：

`cp Makefile.config.example Makefile.config` 

### 指定 opencv 版本

在 Makefile.config 中第 22 行，如果所安装的 opencv 的版本是 3.x, 则取消改行注释。

### 指定 g++ 路径

在 Makefile.config 中第 27 行，如果没有修改 gcc 和 g++ 的软连接，则取消该行的注释并指定 g++ 路径：

`CUSTOM_CXX := /usr/bin/g++-6`

该路径可以首先通过命令

`whereis g++`

查看原本指定的 g++ 路径，然后在同一个文件夹下找到其他版本的 g++.

### 指定 CUDA 路径

第 30 行，通常不需要修改，可以检查自己的 CUDA 路径是否与之一致

`whereis cuda`

根据文件说明，在第 39 ~ 47 行注释相关语句


### 指定 python 路径

在配置文件中第 71, 72 行，修改 python 路径。

`/opt/conda/bin/python`

### 按照自己需求修改其他配置

按照自己的需求修改配置，如使用 python 接口与使用 anaconda 等。

### 指定 hdf5 文件夹

首先找到：

`INCLUDE_DIRS := $(PYTHON_INCLUDE) /usr/local/include`

在其后加上：`/usr/include/hdf5/serial/`

## 编译源码

`# Adjust Makefile.config (for example, if using Anaconda Python, or if cuDNN is desired)`

`make all`

`make test`

`make runtest`

### 暂未解决的问题

已经成功执行以下命令：

`make all`

`make test`

然而执行 `make runtest` 时，遇到以下错误

```
F1015 09:25:16.522922 29408 math_functions.cu:62] Check failed: status == CUBLAS_STATUS_SUCCESS (13 vs. 0)  CUBLAS_STATUS_EXECUTION_FAILED
*** Check failure stack trace: ***
    @     0x7fe66e3fd0cd  google::LogMessage::Fail()
    @     0x7fe66e3fef33  google::LogMessage::SendToLog()
    @     0x7fe66e3fcc28  google::LogMessage::Flush()
    @     0x7fe66e3ff999  google::LogMessageFatal::~LogMessageFatal()
    @     0x7fe66bf4dcf2  caffe::caffe_gpu_gemv<>()
    @     0x7fe66bf15447  caffe::ContrastiveLossLayer<>::Forward_gpu()
    @     0x55d75dd8ed68  caffe::Layer<>::Forward()
    @     0x55d75ddb67b1  caffe::GradientChecker<>::CheckGradientSingle()
    @     0x55d75ddb73e3  caffe::GradientChecker<>::CheckGradientExhaustive()
    @     0x55d75de1ffd1  caffe::ContrastiveLossLayerTest_TestGradient_Test<>::TestBody()
    @     0x55d75e1dd1b4  testing::internal::HandleExceptionsInMethodIfSupported<>()
    @     0x55d75e1d67ba  testing::Test::Run()
    @     0x55d75e1d6908  testing::TestInfo::Run()
    @     0x55d75e1d69e5  testing::TestCase::Run()
    @     0x55d75e1d6ca7  testing::internal::UnitTestImpl::RunAllTests()
    @     0x55d75e1d6fb3  testing::UnitTest::Run()
    @     0x55d75dd8093d  main
    @     0x7fe66b275b97  __libc_start_main
    @     0x55d75dd8840a  _start
make: *** [Makefile:544: runtest] Aborted (core dumped)
```