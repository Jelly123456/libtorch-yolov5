## Introduction

Inspired by https://github.com/yasenh/libtorch-yolov5 and adding several improvements for my own business use case.


A LibTorch inference implementation of the [yolov5](https://github.com/ultralytics/yolov5) object detection algorithm. Both GPU and CPU are supported.

This will enable the AI scientist can train the model with pytorch and deploy the model in C++ by adding the flexibility.



## Dependencies

- Ubuntu 16.04
- CUDA 10.2
- OpenCV 4.1.0
- LibTorch 1.6.0

## Some specail notes:
- GPU inference is only supported in linux platform, not supported in Windows. There is a bug in libtorch 1.6. https://github.com/yf225/pytorch-cpp-issue-tracker/issues/378
- Exported torchscript weights need to be consistent when executing inference. If the weight is exported in linux, then it can only be parsed in linux when inferencing. Also if it is exported in CPU, then the inference can only be run with CPU. The same rules apply to GPU or multiple GPUs.
- When exporting in python, the image size and batch number can also affect the results in inference phase. These two parameters need to be consisitent between training and inference. In this project, please set "image size" to 640 and "batch size" to 1 because the inference code does not add the flexibility to accept other parameter matrix.
- Libtorch requires g++ to support C++14 standard. Check g++ version before running.


## To do list:
- Modify the inference code to support multiple GPUS

## TorchScript Model Export

Please refer to the official document here: https://github.com/ultralytics/yolov5/issues/251



**Mandatory Update**: developer needs to modify following code from the original [export.py in yolov5](https://github.com/ultralytics/yolov5/blob/master/models/export.py)

```bash
# line 29
model.model[-1].export = False
```



**Add GPU support**: Note that the current export script in [yolov5](https://github.com/ultralytics/yolov5) **uses CPU by default**,  the "export.py" needs to be modified as following to support GPU:

```python
# line 28
img = torch.zeros((opt.batch_size, 3, *opt.img_size)).to(device='cuda')  
# line 31
model = attempt_load(opt.weights, map_location=torch.device('cuda'))
```

**Multiple GPU support**: The below example is to use cuda device 0. Feel free to change the nuber inside cuda:0 if you want another cuda device. The command to check cuda device is "nvidia-smi".

```python
# line 28
img = torch.zeros((opt.batch_size, 3, *opt.img_size)).to(device='cuda:0')  
# line 31
model = attempt_load(opt.weights, map_location=torch.device('cuda:0'))
```

Export a trained yolov5 model:

```bash
cd yolov5
export PYTHONPATH="$PWD"  # add path
python models/export.py --weights yolov5s.pt --img 640 --batch 1  # export
```



## Setup

```bash
$ cd /path/to/libtorch-yolo5
$ wget https://download.pytorch.org/libtorch/cu102/libtorch-cxx11-abi-shared-with-deps-1.6.0.zip
$ unzip libtorch-cxx11-abi-shared-with-deps-1.6.0.zip
$ mkdir build && cd build
$ cmake .. or cmake .. -DCMAKE_C_COMPILER="your gcc compiler path" -DCMAKE_CXX_COMPILER="your g++ compiler path" 
$ make
```



To run inference on examples in the `./images` folder:

```bash
# CPU
$ ./libtorch-yolov5 --source ../images/bus.jpg --weights ../weights/yolov5s.torchscript.pt --view-img
# GPU
$ ./libtorch-yolov5 --source ../images/bus.jpg --weights ../weights/yolov5s.torchscript.pt --gpu --view-img
# Profiling
$ CUDA_LAUNCH_BLOCKING=1 ./libtorch-yolov5 --source ../images/bus.jpg --weights ../weights/yolov5s.torchscript.pt --gpu --view-img
```



## Demo

![Bus](images/bus_out.jpg)



![Zidane](images/zidane_out.jpg)



## FAQ

1. terminate called after throwing an instance of 'c10::Error' what(): isTuple() INTERNAL ASSERT FAILED
   
- Make sure "model.model[-1].export = False" when running export script.
   
2. Why the first "inference takes" so long from the log?

   - The first inference is slower as well due to the initial optimization that the JIT (Just-in-time compilation) is doing on your code. This is similar to "warm up" in other JIT compilers. Typically, production services will warm up a model using representative inputs before marking it as available.

   - It may take longer time for the first cycle. The [yolov5 python version](https://github.com/ultralytics/yolov5) run the inference once with an empty image before the actual detection pipeline. User can modify the code to process the same image multiple times or process a video to get the valid processing time.



## References

1. https://github.com/ultralytics/yolov5
2. https://github.com/walktree/libtorch-yolov3
3. https://pytorch.org/cppdocs/index.html
4. https://github.com/pytorch/vision
5. [PyTorch.org - CUDA SEMANTICS](https://pytorch.org/docs/stable/notes/cuda.html)
6. [PyTorch.org - add synchronization points](https://discuss.pytorch.org/t/why-is-the-const-time-with-fp32-and-fp16-almost-the-same-in-libtorchs-forward/45792/5)
7. [PyTorch - why first inference is slower](https://github.com/pytorch/pytorch/issues/2694)

