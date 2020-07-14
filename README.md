# 在CORE-3399PRO-JD4上编译安装pytorch和opencv

最近拿到一块[CORE-3399PRO-JD4](http://wiki.t-firefly.com/zh_CN/Core-3399pro-JD4/started.html)开发板，要在上面部署AI应用。这块板子搭载Rockchip RK3399Pro处理器，采用双核 Cortex-A72+四核 Cortex-A53构架，主频高达 1.8GHz，集成四核Mali-T864 GPU，并内置高性能NPU，号称性能优异。但是实际过程中，发现发热严重，要使用NPU需要把算法移植到Rockchip提供的RKNN-Toolkit开发套件。按照[官方文档](http://wiki.t-firefly.com/zh_CN/3399pro_npu/npu_rknn_toolkit.html)把板子刷成了ubuntu18.4，然后安装了RKNN，打算把现有模型转换一下，结果发现： 

- 适配的tensorflow版本较低（RKNN 1.3 支持tensorflow 1.10.1），2.0版本的模型无法转换。
- ONNX版本模型转换失败，应该也是版本不对
- pytorch干脆没有迁移到板子上，官方例程都跑不起来。



实在没办法，只能先不用NPU把应用跑起来试试。这里把编译在ARM Ubuntu18.04上编译pytorch和opencv的过程记录一下。

## 编译pytorch

在开发板上编译，需要先安装编译工具链（要联网）。因为开始编译前我已经折腾这块板子好多天了，下面列出的依赖可能不是全部。

```bash
sudo apt-get install libopenblas-dev libblas-dev m4 cmake cython
```

然后安装python依赖，我这里使用的是python3.6。

```bash
pip3 install numpy pyyaml cyphon
```

从gitbub拿下来pytorch源码，submodule update 过程必不可少。

```bash
git clone --recursive https://github.com/pytorch/pytorch
cd pytorch
git checkout v1.4
git submodule sync
git submodule update --init --recursive
git submodule update --remote third_party/protobuf
```

配置环境变量

```bash
export NO_CUDA=1          #不适用cuda
export NO_DISTRIBUTED=1   #不支持分布式
export NO_MKLDNN=1        #不支持MKLDNN
export MAX_JOBS=4 
```

需要注意的是MAX_JOBS是并行编译的最大线程数。（虽然CORE-3399PRO-JD4 这块板子是6核CPU，可以支持6线程编译，但是它只有4GB内存，编译一些复杂模块时，GCC会耗尽内存崩溃。我的经验是先用4线程或6线程编译，遇到GCC崩溃，再改回单线程编译，耗内存的模块编译过之后再改回6线程编译。）


开始编译

```bash
#打包成whl，打包成功后这个文件在dist目录里面
python setup.py bdist_wheel
```

安装编译好的wheel包

```bash
cd dist
pip3 install ./torch-1.4.0a0+72e1771-cp36-cp36m-linux_aarch64.whl
```


## **编译torchvision**

先安装编译依赖

```bash
sudo apt-get install libjpeg-dev libavcodec-dev libavformat-dev libswscale-dev
pip3 install pillow
```

从gitbub clone 代码

```bash
git clone https://github.com/pytorch/vision.git
cd vision
git checkout v0.5
```

开始编译

```bash
#打包成whl
python setup.py bdist_wheel
```

安装

```bash
cd dist
pip3 install ./torchvision-0.5.0a0+85b8fbf-cp36-cp36m-linux_aarch64.whl
```


## 编译OpenCV

ubuntu 18.04 可以直接用 `apt` 安装opencv 3.2，但是我们之前的一个应用至少需要3.3版本，所以也需要重新编译。
首先安装工具链

```bash
sudo apt-get install build-essential cmake git libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev
```

下载OpenCV和opencv_conrib 源码

```bash
wget -O opencv-3.3.0.zip https://github.com/opencv/opencv/archive/3.3.0.zip
wget -O opencv_contrib.zip https://github.com/opencv/opencv_contrib/archive/3.3.0.zip
unzip opencv-3.3.0.zip
unzip opencv_contrib.zip
cd opencv-3.3.0
```

配置cmake

```bash
mkdir build
cd build
cmake -D CMAKE_BUILD_TYPE=RELEASE \
      -D CMAKE_INSTALL_PREFIX=/usr/local \
      -D INSTALL_PYTHON_EXAMPLES=ON \
      -D INSTALL_C_EXAMPLES=OFF \
      -D OPENCV_EXTRA_MODULES_PATH=/path/to/opencv_contrib-3.3.0/modules \
      -D PYTHON_EXECUTABLE=/usr/bing/python3 \
      -D BUILD_EXAMPLES=ON ..
```

开始编译

```bash
make -j 4
```

打包和安装

```bash
make package
sudo make install
```

## 软件包下载

由于是在开发板上编译，PyTorch花费了大概4个小时，OpenCV也需要1个小时。我把编译好的包上传到了github，有需要的可以直接下载：

- [OpenCV-3.3.0](https://github.com/yunfeizu/RK3399Pro-pytorch-opencv/tree/master/opencv)
- [PyTorch-1.4.0-python3.6](https://github.com/yunfeizu/RK3399Pro-pytorch-opencv/tree/master/pytorch)
