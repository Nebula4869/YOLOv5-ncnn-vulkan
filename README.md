# YOLOv5 ncnn-vulkan
An ncnn-vulkan implementation of YOLOv5, capable of using GPU to accelerate inference

### Environment

- Ubuntu 18.04
- OpenCV 3.2.0
- CMake 3.10.2

### Getting Started

1. Install OpenCV.

   ```shell
   sudo apt-get install libopencv-dev
   ```

2. Edit "CMakeLists.txt" to configure OpenCV correctly.

4. Compile and run.

   ```shell
   cd build
   cmake ..
   make
   ./../bin/YOLOv5ncnn-vulkan
   ```

### Get your own Yolov5 ncnn model

We train a model in Pytorch and first convert to onnx and then to ncnn.

1. For how to train in Pytorch and export to onnx, see https://github.com/ultralytics/yolov5.

2. Because ncnn has limited support for operators, the network definition needs to be modified before training, please modify "common.py".

   from

   ```python
   class Focus(nn.Module):
       def __init__(self, c1, c2, k=1, s=1, p=None, g=1, act=True):
           super(Focus, self).__init__()
           self.conv = Conv(c1 * 4, c2, k, s, p, g, act)
   
       def forward(self, x):
           return self.conv(torch.cat([x[..., ::2, ::2], x[..., ::2, ::2], x[..., ::2, ::2], x[..., ::2, ::2]], 1))
   ```

   to

   ```python
   class Focus(nn.Module):
       def __init__(self, c1, c2, k=1, s=1, p=None, g=1, act=True):
           super(Focus, self).__init__()
           self.conv = Conv(c1 * 4, c2, k, s, p, g, act)
   
       def forward(self, x):
           return self.conv(torch.cat([torch.nn.functional.interpolate(x, scale_factor=0.5),
                                       torch.nn.functional.interpolate(x, scale_factor=0.5),
                                       torch.nn.functional.interpolate(x, scale_factor=0.5),
                                       torch.nn.functional.interpolate(x, scale_factor=0.5)], 1))
   ```

3. When export to onnx, Detect layer should be removed from the graph, please modify  "export.py".

   ```python
   model.model[-1].export = True
   ```

4. Simplify the onnx model by onnx-simplifier.

   ```shell
   pip3 install onnx-simplifier
   python3 -m onnxsim yolov5s.onnx yolov5s.onnx
   ```

5. Convert onnx to ncnn

   ```shell
   ./onnx2ncnn yolov5s.onnx yolov5s.param yolov5s.bin
   ```

### Compile ncnn-vulkan by yourself

The executable file onnx2ncnn and the files in the lib and include directories come from the compiled ncnn and vulkan. If you want to compile it yourself:

1. Install protobuf.

   ```shell
   sudo apt install protobuf-compiler libprotobuf-dev 
   ```

2. Download vulkan-sdk from https://vulkan.lunarg.com/sdk/home#sdk/downloadConfirm/1.2.148.0/linux/vulkansdk-linux-x86_64-1.2.148.0.tar.gz. and add to environment variables (reboot may needed).

   ```shell
   export VULKAN_SDK=~/vulkan-sdk-1.2.148.0/x86_64
   export PATH=$PATH:$VULKAN_SDK/bin:
   export LIBRARY_PATH=$LIBRARY_PATH$:VULKAN_SDK/lib
   export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$VULKAN_SDK/lib
   export VK_LAYER_PATH=$VULKAN_SDK/etc/vulkan/explicit_layer.d
   ```

3. Download source code of ncnn from https://github.com/Tencent/ncnn/releases.

   ```shell
   unzip ncnn-master.zip
   ```

4. Compile ncnn.

   ```shell
   mkdir build
   cd build
   cmake -DCMAKE_TOOLCHAIN_FILE=../toolchains/host.gcc.toolchain.cmake -DNCNN_VULKAN=ON ..
   make -j8
   make install
   ```
