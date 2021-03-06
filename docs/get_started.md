# 安装和使用教程

## 环境需求

```yaml
python>=3.6
numpy>=1.16.0
torch>=1.0  # effdet需要torch>=1.5，如果不使用effdet，在network/__init__.py下将其注释掉
tensorboardX>=1.6
utils-misc>=0.0.5
mscv>=0.0.3
matplotlib>=3.1.1
opencv-python>=4.2.0.34  # opencv>=4.4版本需要编译，耗时较长，建议安装4.2版本
opencv-python-headless>=4.2.0.34
albumentations>=0.5.1  # 需要opencv>=4.2
scikit-image>=0.17.2
easydict>=1.9
timm==0.1.30  # timm >= 0.2.0 不兼容 
typing_extensions==3.7.2
tqdm>=4.49.0
PyYAML>=5.3.1
Cython>=0.29.16
pycocotools>=2.0  # 需要Cython
omegaconf>=2.0.0  # effdet依赖

```

　　都是很好装的包，使用`pip install`安装即可，不需要编译(最好逐行安装，因为存在前后依赖关系)。

## 训练和验证模型voc数据集

### 克隆此项目

```bash
git clone https://github.com/misads/detection_template
cd detection_template
```

### 准备voc数据集

1. 在项目目录下新建`datasets`目录：

   ```bash
   mkdir datasets
   ```

2. 将voc数据集的`VOC2007`或者`VOC2017`目录移动`datasets/voc`目录。（推荐使用软链接）

   ```bash
   ln -s <VOC的下载路径>/VOCdevkit/VOC2017 datasets/voc
   ```

3. 数据准备好后，数据的目录结构看起来应该是这样的：

   ```yml
   detection_template
       └── datasets
             ├── voc           
             │    ├── Annotations
             │    ├── JPEGImages
             │    └── ImageSets/Main
             │            ├── train.txt
             │            └── test.txt
             └── <其他数据集>
   ```

### 验证模型


1. 新建`pretrained`文件夹：

   ```bash
   mkdir pretrained
   ```

2. 以Faster-RCNN为例，下载[[预训练模型]](https://github.com/misads/detection_template#%E9%A2%84%E8%AE%AD%E7%BB%83%E6%A8%A1%E5%9E%8B)，并将其放在`pretrained`目录下：

   ```yml
   detection_template
       └── pretrained
             └── 0_voc_FasterRCNN.pt
   ```

3. 运行以下命令来验证模型的`mAP`指标：

   ```bash
   python3 eval.py --model Faster_RCNN --dataset voc --load pretrained/0_voc_FasterRCNN.pt -b1
   ```

4. 如果需要使用`Tensorboard`可视化预测结果，可以在上面的命令最后加上`--vis`参数。然后运行`tensorboard --logdir results/cache`查看检测的可视化结果。

4. 使用其他的模型只需要修改`--model`参数即可。

### 训练模型

#### Faster RCNN

```bash
python3 train.py --tag frcnn_voc --model Faster_RCNN -b1 --optimizer sgd --val_freq 1 --save_freq 1 --epochs 12 --lr 1
```

#### YOLOv2

```bash
python3 train.py --tag yolo2_voc --model Yolo2 -b24 --val_freq 5 --save_freq 5 --optimizer sgd --lr 1. --weights pretrained/darknet19_448.conv.23 --epochs 160
```

`darknet19_448.conv.23`是Yolo2在`ImageNet`上的预训练模型，可以在yolo官网下载。[[下载地址]](https://pjreddie.com/media/files/darknet19_448.conv.23)。

### 参数说明

`--tag`参数是一次操作(`train`或`eval`)的标签，日志会保存在`logs/标签`目录下，保存的模型会保存在`checkpoints/标签`目录下。  

`--model`是使用的模型，所有可用的模型定义在`network/__init__.py`中。  

`--epochs`是训练的代数。  

`-b`参数是`batch_size`，可以根据显存的大小调整。  

`-w`参数是`num_workers`，即读取数据的进程数，如果需要用pdb来debug，将这个参数设为0。  

`--lr`是初始学习率(如果使用`lambda`自定义学习率，`—lr`设为1即可)。

`--load`是加载预训练模型。  

`--resume`配合`--load`使用，会恢复上次训练的`epoch`和优化器。  

`--val_freq`和`--save_freq`分别是每几代验证一次和每几代保存一次`checkpoint`。

## 训练和验证模型自定义数据集(voc格式)

### 准备数据集

1. 将自己的数据集制作成`VOC`格式，并放在`datasets`目录下(可以使用软链接)。目录结构如下：

   ```yml
   detection_template
       └── datasets
             └── mydata_dir    
                  ├── Annotations
                  ├── JPEGImages
                  └── ImageSets/Main
                          ├── train.txt
                          └── val.txt
   ```

2. 在`dataloader/custom`目录下新建一个`my_data.py`，内容如下：

   ```python
   class MyData(object):
       data_format = 'VOC'
       voc_root = 'datasets/mydata_dir'
       train_split = 'train.txt'
       val_split = 'val.txt' 
       class_names = ["car", "person", "bus"]  # 所有的类别名
       
       img_format = 'jpg'  # 根据图片文件是jpg还是png设为'jpg'或者'png'
   ```

3. 在`dataloader/custom/__init__.py`中添加自己的数据集类：

   ```python
   from .origin_voc import VOC  # 这是原始的VOC
   from .my_data import MyData
   
   datasets = {
       'voc': VOC,  # if --dataset is not specified
       'mydata': MyData
   }
   ```

4. 完成定义数据集后，训练和验证时就可以使用`--dataset mydata`参数来使用自己的数据集。

### 预览数据集标注

1. 运行`preview.py`：

   ```bash
   python3 preview.py --dataset mydata --transform none
   ```

2. 运行`tensorboard`：

   ```bash
   tensorboard --logdir logs/preview
   ```

3. 打开浏览器，查看标注是否正确。

### 在自定义数据及上训练已有模型

#### Faster RCNN

```bash
python3 train.py --tag frcnn_mydata --dataset mydata --model Faster_RCNN -b1 --optimizer sgd --val_freq 1 --save_freq 1 --epochs 12 --lr 1
```

#### YOLOv2

```bash
python3 train.py --tag yolo2_mydata --dataset mydata --model Yolo2 -b24 --val_freq 5 --save_freq 5 --optimizer sgd --lr 1. --weights pretrained/darknet19_448.conv.23 --epochs 160
```

## 添加新的检测模型

1. 复制`network`目录下的`Faster_RCNN`文件夹，改成另外一个名字(比如`MyNet`)。
2. 仿照`Faster_RCNN`的model.py，修改自己的网络结构、损失函数和优化过程。
3. 在`network/__init__.py`中`import`你模型的`Model`类并且在`models = {}`中添加它。

```python
from .Faster_RCNN.Model import Model as Faster_RCNN
from .MyNet.Model import Model as MyNet
models = {
    'Faster_RCNN': Faster_RCNN,
    'MyNet': MyNet,
}
```

4. 运行 `python train.py --model MyNet` 看能否成功运行

