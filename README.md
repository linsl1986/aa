# 基于PaddleX的YOLOv3废水水质判断


基于PaddleX的YOLOv3进行废水水质判断，包括自制数据集、数据加载和预处理、模型选择和调参、模型部署等步骤，同时总结了心得体会。

# 一、项目背景

利用无人机巡查河道，及时发现污染源是目前地表水监测的热点。通过搭载在无人机上的摄像头，手机上可以实时查看河道的水体图像。因此需要部署在手机端通过目标检测项目来实时判断图像中的水质是否达标。如果发现水质超标，则可以通过无人机进一步寻找污染源，以提升地表水监测能力。
![](https://ai-studio-static-online.cdn.bcebos.com/01d77919263d4eb7bc518d617d590fdcf7b8db9e4d4e42e594332294a2dd95b5)
![](https://ai-studio-static-online.cdn.bcebos.com/a44290dd46df47f091281b93436c058140f6f2a6927b4bc2b8d34e179834cc66)

本项目基于PaddleX的YOLOv3进行废水水质判断，


![](https://ai-studio-static-online.cdn.bcebos.com/5a3010ff34d8471b81cce9ca3fe58132dc55fff92ad544daa9e5fe0437e62e25)


# 二、数据集简介

本项目从网页上收集了40个水质样品图片，然后使用labelImg对图片进行标注。labelImg的安装及使用请参见https://blog.csdn.net/python_pycharm/article/details/85338801

## 1.自制VOC数据集
在home目录下上传包含图片文件和标注文件的压缩包facemask.zip（考虑到尽量不修改老师源代码，因此连压缩包名字都与“口罩检测”项目一致，这样简化了代码修改），图片文件储存于JPEGImages，标注文件储存于Annotations。然后解压文件，具体代码如下：


```python
#下载工具包，不用每次运行。使用github替代gitee也行，但很慢。
!git clone https://gitee.com/PaddlePaddle/PaddleDetection.git

#存入持久层
!mv PaddleDetection/ work/

#安装运行环境，每次打开都需要运行
!pip install -r work/PaddleDetection/requirements.txt

#解压自制数据集
!unzip -oq /home/aistudio/facemask.zip -d work/PaddleDetection/dataset/MaskVOCData

#安装paddleX
!pip install paddlex
#对数据集进行划分，注意train_value+val_value+test_value=1，因此代码中只需要标注评估集数据比例val_value和测试集数据比例test_value
!paddlex --split_dataset --format VOC --dataset_dir work/PaddleDetection/dataset/MaskVOCData/ --val_value 0.05  --test_value 0.15
#经过上述步骤，MaskVOCData文件夹多了labels.txt，test_list.txt，train_list.txt以及val_list.txt

%cd work/PaddleDetection/
#制作VOC数据集
#提取文件下img目录所有照片名不要后缀
import pandas as pd 
import os

filelist = os.listdir("dataset/MaskVOCData/JPEGImages")
train_name = []

for file_name in filelist:
    name, point ,end =file_name.partition('.')
    train_name.append(name)

df = pd.DataFrame(train_name) 
df.head(8)

df.to_csv('./train_all.txt', sep='\t', index=None,header=None) 

!mkdir -p dataset/MaskVOCData/ImageSets/Main #创建多级目录
!mv train_all.txt dataset/MaskVOCData/ImageSets
!mv dataset/MaskVOCData/labels.txt dataset/MaskVOCData/label_list.txt
!cp dataset/MaskVOCData/label_list.txt dataset/MaskVOCData/ImageSets/

#备份VOC 数据集到home目录下
!cp -r dataset/MaskVOCData  /home/aistudio/
! tar   -cvf     /home/aistudio/MaskVOCData.tar     /home/aistudio/MaskVOCData/  

#以下开始制作COCO数据集
!python tools/x2coco.py \
        --dataset_type voc \
        --voc_anno_dir dataset/MaskVOCData/Annotations \
        --voc_anno_list dataset/MaskVOCData/ImageSets/train_all.txt \
        --voc_label_list dataset/MaskVOCData/ImageSets/label_list.txt \
        --voc_out_name ./dataset/annotations.json

!mv dataset/MaskVOCData dataset/MaskCOCOData
!mv ../../MaskVOCData dataset
!mkdir dataset/MaskCOCOData/annotations
!mv dataset/annotations.json  dataset/MaskCOCOData/annotations
!rm dataset/MaskCOCOData/train_list.txt
!rm dataset/MaskCOCOData/val_list.txt
!rm dataset/MaskCOCOData/label_list.txt
!rm dataset/MaskCOCOData/test_list.txt
!rm -r dataset/MaskCOCOData/Annotations
!rm -r dataset/MaskCOCOData/ImageSets

!paddlex --split_dataset --format COCO --dataset_dir dataset/MaskCOCOData/annotations --val_value 0.05  --test_value 0.15

#备份COCO数据集到home目录下
!cp -r dataset/MaskCOCOData  /home/aistudio/
! tar   -cvf     /home/aistudio/MaskCOCOData.tar     /home/aistudio/MaskCOCOData/ 
```




## 2.数据加载和预处理
主要任务为图像数据增强，为后续训练提供基础

```python
%cd /home/aistudio/work/PaddleDetection
import paddle
import paddlex as pdx
import numpy as np
import paddle.nn as nn
import paddle.nn.functional as F
import PIL.Image as Image
import cv2 
import os

from random import shuffle
from paddlex.det import transforms as T
from PIL import Image, ImageFilter, ImageEnhance
import matplotlib.pyplot as plt # plt 用于显示图片

#这部分T是数据增强，不是全部使用才能提高map，只能选择合适的才能提高，所以需要调整，调整时就在原来的句子前加#

def preprocess(dataType="train"):
    if dataType == "train":
        transform = T.ComposedYOLOv3Transforms([
            T.MixupImage(mixup_epoch=250),   #对图像进行mixup操作，模型训练时的数据增强操作，目前仅YOLOv3模型支持该transform
             T.RandomExpand(),        #随机扩张图像
             T.RandomDistort(brightness_range=1.2, brightness_prob=0.3),       #以一定的概率对图像进行随机像素内容变换
             T.RandomCrop(),          #随机裁剪图像
             T.ResizeByShort(),  #根据图像的短边调整图像大小
            T.Resize(target_size=608, interp='RANDOM'),   #调整图像大小,[’NEAREST’, ‘LINEAR’, ‘CUBIC’, ‘AREA’, ‘LANCZOS4’, ‘RANDOM’]
             T.RandomHorizontalFlip(),  #以一定的概率对图像进行随机水平翻转
            T.Normalize([0.485, 0.456, 0.406],[0.229, 0.224, 0.225])  #对图像进行标准化(mean,std)
            ])



        return transform
    else:
        transform = T.Compose([
            T.Resize(target_size=608, interp='RANDOM'), 
            T.Normalize([0.485, 0.456, 0.406],[0.229, 0.224, 0.225])
            ])
        return transform


train_transforms = preprocess(dataType="train")
eval_transforms  = preprocess(dataType="eval")



# 定义训练和验证所用的数据集
# API地址：https://paddlex.readthedocs.io/zh_CN/develop/data/format/detection.html?highlight=paddlex.det
train_dataset = pdx.datasets.VOCDetection(
    data_dir='./dataset/MaskVOCData',
    file_list='./dataset/MaskVOCData/train_list.txt',
    label_list='./dataset/MaskVOCData/label_list.txt',
    transforms=train_transforms,
    shuffle=True)
eval_dataset = pdx.datasets.VOCDetection(
    data_dir='./dataset/MaskVOCData',
    file_list='./dataset/MaskVOCData/val_list.txt',
    label_list='./dataset/MaskVOCData/label_list.txt',
    transforms=eval_transforms)



```
# 三、模型选择和调参

使用yolov3模型，这里使用MobileNetV1网络，便于部署在移动端，如下所示：

## 1.模型选择


```python
#使用yolov3模型，具体信息可以查看work\PaddleDetection\configs里面的文件，这里使用MobileNetV1网络，便于部署在移动端
import matplotlib
matplotlib.use('Agg') 
os.environ['CUDA_VISIBLE_DEVICES'] = '0'#0为GPU模式
%matplotlib inline
import warnings
warnings.filterwarnings("ignore")

 #可使用VisualDL查看训练指标，参考https://paddlex.readthedocs.io/zh_CN/develop/train/visualdl.html
num_classes = len(train_dataset.labels)

# API说明: https://paddlex.readthedocs.io/zh_CN/release-1.3/appendix/model_zoo.html
model = pdx.det.YOLOv3(num_classes=num_classes, backbone='MobileNetV1')


```

## 2.模型参数调整
调整参数，包括训练轮数等，以提高训练效果

```python
# 各参数介绍与调整说明：https://paddlex.readthedocs.io/zh_CN/release-1.3/appendix/parameters.html
model.train(
    num_epochs=260,
    train_dataset=train_dataset,
    train_batch_size=8,
    eval_dataset=eval_dataset,
    learning_rate=0.000125,
    warmup_steps=800, #默认值为1000，但代码运行时提示“warmup_steps should less than 852 or lr_decay_epochs[0] greater than 250, please modify 'lr_decay_epochs' or 'warmup_steps' in train function”
    warmup_start_lr=0.0,
    save_interval_epochs=20,
    lr_decay_epochs=[213, 240],
    lr_decay_gamma=0.1,
    save_dir='output/yolov3_MobileNetV1',
    use_vdl=True)
```
    


## 3.模型评估测试记录最优模型参数
模型在训练（对应train）时会自动输出每个循环的loss（损失函数）和相应的lr（学习率），
每20步（因为我设置了20步记录一次数据）就会评估（对应eval）计算bbox_map来判断模型的优劣。
同时程序会自动记录bbox_map大的情况相应的参数，作为best_model（存在于/home/aistudio/work/PaddleDetection/output/yolov3_MobileNetV1/best_model）
```
2021-08-12 14:24:51 [INFO]	Decompressing output/yolov3_MobileNetV1/pretrain/MobileNetV1_pretrained.tar...
2021-08-12 14:24:56 [INFO]	Load pretrain weights from output/yolov3_MobileNetV1/pretrain/MobileNetV1_pretrained.
2021-08-12 14:24:56 [INFO]	There are 135 varaibles in output/yolov3_MobileNetV1/pretrain/MobileNetV1_pretrained are loaded.
2021-08-12 14:25:01 [INFO]	[TRAIN] Epoch=1/260, Step=2/4, loss=19905.947266, lr=0.0, time_each_step=2.13s, eta=0:37:18
。。。。。。。
2021-08-12 14:42:16 [INFO]	[TRAIN] Epoch 260 finished, loss=5.838893, lr=1e-06 .
2021-08-12 14:42:16 [INFO]	Start to evaluating(total_samples=2, total_steps=1)...
100%|██████████| 1/1 [00:05<00:00,  5.53s/it]
2021-08-12 14:42:22 [INFO]	[EVAL] Finished, Epoch=260, bbox_map=100.0 .
2021-08-12 14:42:22 [INFO]	Model saved in output/yolov3_MobileNetV1/epoch_260.
2021-08-12 14:42:22 [INFO]	Current evaluated best model in eval_dataset is epoch_140, bbox_map=100.0
```

## 4.模型预测

### 4.1 批量预测

使用model.predict接口来完成对大量数据集的批量预测。



```python
#解压测试文件
!unzip -oq /home/aistudio/Test.zip -d /home/aistudio/Test

#进行批量预测
%cd /home/aistudio/work/PaddleDetection
model = pdx.load_model('output/yolov3_MobileNetV1/best_model')

image_dir = '/home/aistudio/Test'
images = os.listdir(image_dir)

for img in images:
   image_name = image_dir +"/" +img 
   result = model.predict(image_name)   # 进行预测操作
   pdx.det.visualize(image_name, result, threshold=0.03, save_dir='/home/aistudio/Result') #定义预测结果输入参数，其中threshold为预测精度临界值，只有高于此值才显示预测结果

#展示模型推理结果
image_dir = '/home/aistudio/Test'
images = os.listdir(image_dir)

for img in images:
   image_name = image_dir +"/" +img 
   yuanshi = Image.open(image_name)
   plt.imshow(yuanshi)          #根据数组绘制图像
   plt.show()               #显示图像
   yuce=Image.open('/home/aistudio/Result/visualize_'+img)
   plt.imshow(yuce)          #根据数组绘制图像
   plt.show()               #显示图像
```

## 5.模型输出

使用PaddleDetection的工具导出模型。
然后使用 paddlelite把模型输出成nb文件
最后下载work/PaddleDetection/inference文件以及work/PaddleDetection/dataset/MaskVOCData文件，然后解压备用


```python

%cd /home/aistudio/work/PaddleDetection/

# 导出模型，可以参考/home/aistudio/work/PaddleDetection/configs/里面的yml文件，看看哪个模型更合适就导出成哪种模型
!python tools/export_model.py -c configs/yolov3/yolov3_mobilenet_v1_270e_voc.yml -o weights=output/yolov3_MobileNetV1/best_model/model.pdparams --output_dir ./inference

# 准备PaddleLite依赖
!pip install paddlelite

%cd /home/aistudio/work/PaddleDetection/

# 准备PaddleLite部署模型
#--valid_targets中参数（arm）用于传统手机，（npu,arm ）用于华为带有npu处理器的手机
!paddle_lite_opt \
    --model_file=inference/yolov3_mobilenet_v1_270e_voc/model.pdmodel \
    --param_file=inference/yolov3_mobilenet_v1_270e_voc/model.pdiparams \
    --optimize_out=./inference/yolov3_mobilenet_v1_270e_voc \
    --optimize_out_type=naive_buffer \
    --valid_targets=arm 
    #--valid_targets=npu,arm 
```

## 6.安卓端部署
6.1安装Android Studio
登录https://developer.android.google.cn/studio/
![](https://ai-studio-static-online.cdn.bcebos.com/66a42343e04748389bca8c6a53f39820f6ef0895c74b4bb2aafaadb35d620e36)

按步骤一直安装下去

6.2下载Paddle-Lite-Demo
登录https://gitee.com/paddlepaddle/Paddle-Lite-Demo
![](https://ai-studio-static-online.cdn.bcebos.com/6d85ea9223774dee9ba22a69fdc617cb536f4bbb866c4c2dadf54afb4cd09c0b)

解压到自己喜欢的文件夹下

6.3 打开Demo
![](https://ai-studio-static-online.cdn.bcebos.com/5eb96fa681eb456cb3ab3e20a4c4a90cfee208e2de8a4219886852f8251caedc)

6.4 修改系列参数
1、在assets\models目录下新建mask目录，并存放解压出来的\home\aistudio\work\PaddleDetection\inference\yolov3_mobilenet_v1_270e_voc.nb，并且改名成model.nb

![](https://ai-studio-static-online.cdn.bcebos.com/2be2c5bd356241d68daa1ac3ade75da992642f3d9278435483a2a89960be41fa)

2、将解压出来的\home\aistudio\work\PaddleDetection\dataset\MaskVOCData\labels.txt拉到assets\labels目录下

![](https://ai-studio-static-online.cdn.bcebos.com/57d59c0763854bc0b184805d048d9964830e04b705e64865b30c2cb843f8f0b8)

修改labels.txt内容

![](https://ai-studio-static-online.cdn.bcebos.com/c26d96b42a344d9f94c64cec34d30550d90ef60d921b4900ac3a15fd93ed77a3)


3、将解压出来的\home\aistudio\work\PaddleDetection\dataset\MaskVOCData\JPEGImages中一个图片拉到assets\images目录下，作为app的示例图片

![](https://ai-studio-static-online.cdn.bcebos.com/1aaecacaebf543b7a6fa7dc418452e33931856aa3ebf444e97156061bed7a0ce)

4、相应修改res\values\strings.xml参数

![](https://ai-studio-static-online.cdn.bcebos.com/5145d0372eae4edaac495516da0a1abe356a5f6abe71419a8561574a85b28293)


6.5安装app
插入手机，进入“开发者模式”

![](https://ai-studio-static-online.cdn.bcebos.com/583f3ba1953b4d39bd27122aaa0f1a58f233c460d8164c0c81dc3969b6ba6734)

按run app

![](https://ai-studio-static-online.cdn.bcebos.com/c6a18b0ea01648eeb8b2801ffff02acfd785e98ee0fd4f97849c6d2942838e1d)

自动安装

![](https://ai-studio-static-online.cdn.bcebos.com/3c608e4d1ce549e9b1c576f6e6f21590b5cbe0b9025047d996c07ea600f18be4)

![](https://ai-studio-static-online.cdn.bcebos.com/b5c5046dff3a4f60a090f3de163fe9f4532bb5c1e8494b13af077c79117d9e6c)



# 四、效果展示

鉴于最终app制作失败，我只能使用python预测，在aistudio上进行，结果如下。其中有两个图无结果显示。
![](https://ai-studio-static-online.cdn.bcebos.com/70a8c54f743348578c6701f2b124dfbf0b87232cab424d648ab1e23303e630d9)
![](https://ai-studio-static-online.cdn.bcebos.com/9d82f9f23c534b51a2f2358154dab780f7d540596a5843f2a0ccefff647b545e)
![](https://ai-studio-static-online.cdn.bcebos.com/3482239ae0c94724be415a0f71482d1d4d69cddd196a4c948f73478662ef532e)
![](https://ai-studio-static-online.cdn.bcebos.com/86f3d81f3c4e4a90bfcd582fe01b7e62beebc98b7b1b4b8d9581aa681ed4f6e1)



# 五、总结与升华

由于本人从没使用过linux系统，因此完成此项目过程均在win10环境中进行。
主要难点在于与网络上的教程有一定出入，毕竟win10下操作与大神们的ubtune系统很不一样，所以只能吸取大神们的精神要领来完成本项目。
本项目存在问题在于：
1、本项目的预测结果不理想，主要原因是自制数据集中标注没做好，从下图看到，我的标注还是比较随意的。所以即使我努力调参，最终这个识别的准确率还是非常低
![](https://ai-studio-static-online.cdn.bcebos.com/560df1e2bb8e41088044c6d1f1a27301ed490feb5e0844abb5a268f9de35b59b)

2、本项目导出的模型制作的app会闪退（原demo制作的目标检测app运行正常），主要原因是nb文件体积差异太大了，原demo的nb只有，但本项目导出的nb文件有，估计是app运行时占用太大内存了，就挂了。

![](https://ai-studio-static-online.cdn.bcebos.com/c238fb13c76a42028ea02359b96681a38fac0dc756b440a69d3fe180cd53b069)



# 个人简介

我在AI Studio上获得青铜等级，点亮1个徽章，来互关呀~ https://aistudio.baidu.com/aistudio/personalcenter/thirdview/134639
