# 钢筋计数


### 1 项目说明
在该项目中，主要向大家介绍如何使用目标检测来实现对钢筋计数。涉及代码亦可用于车辆计数、螺母计数、圆木计数等。

在工地现场，对于进场的钢筋车，验收人员需要对车上的钢筋进行现场人工点根，确认数量后钢筋车才能完成进场卸货。上述过程繁琐、消耗人力且速度很慢。针对上述问题，希望通过手机拍照->目标检测计数->人工修改少量误检的方式智能、高效的完成此任务：
<div align="center">
<img src="./images/worker.png"  width = "500" />              </div>

**业务难点：**
* **精度要求高** 钢筋本身价格较昂贵，且在实际使用中数量很大，误检和漏检都需要人工在大量的标记点中找出，所以需要精度非常高才能保证验收人员的使用体验。需要专门针对此密集目标的检测算法进行优化，另外，还需要处理拍摄角度、光线不完全受控，钢筋存在长短不齐、可能存在遮挡等情况。
* **钢筋尺寸不一** 钢筋的直径变化范围较大且截面形状不规则、颜色不一，拍摄的角度、距离也不完全受控，这也导致传统算法在实际使用的过程中效果很难稳定。
* **边界难以区分** 一辆钢筋车一次会运输很多捆钢筋，如果直接全部处理会存在边缘角度差、遮挡等问题效果不好，目前在用单捆处理+最后合计的流程，这样的处理过程就会需要对捆间进行分割或者对最终结果进行去重，难度较大。

<div align="center">
<img src="./images/phone_pic.jpg"  width = "1000" />              </div>

### 2 数据准备

数据集中包含了250张已经标注好的数据，原始数据标注形式为csv格式。该项目采用目标检测的标注方式，在本文档中提供了VOC数据集格式。[点击此处下载数据集]( https://bj.bcebos.com/paddlex/examples2/rebar_count/dataset_reinforcing_steel_bar_counting.zip)

更多数据格式信息请参考[数据标注说明文档](https://paddlex.readthedocs.io/zh_CN/develop/data/annotation/index.html)

* **数据切分**
将训练集、验证集和测试集按照7：2：1的比例划分。 PaddleX中提供了简单易用的API，方便用户直接使用进行数据划分。
``` shell
paddlex --split_dataset --format VOC --dataset_dir dataset --val_value 0.2 --test_value 0.1
```
<div align="center">
<img src="./images/split_dataset.png"  width = "1500" />              </div>
数据文件夹切分前后的状态如下:

```bash
  dataset/                          dataset/
  ├── Annotations/      -->         ├── Annotations/
  ├── JPEGImages/                   ├── JPEGImages/
                                    ├── labels.txt
                                    ├── test_list.txt
                                    ├── train_list.txt
                                    ├── val_list.txt
  ```

### 3 模型选择
PaddleX提供了丰富的视觉模型，在目标检测中提供了RCNN和YOLO系列模型。在本项目中采用Faster-RCNN作为检测模型进行钢筋计数。

### 4 模型训练
在项目中，我们采用yolov3作为钢筋检测的模型。具体代码请参考[train.py](./code/train.py)。

运行如下代码开始训练模型：


``` shell
python code/train.py
```

若输入如下代码，则可在log文件中查看训练日志，log文件保存在`code`目标下
``` shell
python code/train.py > log
```

* 训练过程说明
<div align="center">
<img src="./images/process.png"  width = "1000" />              </div>


### 5 模型优化(进阶)
* 精度提升
  为了进一步提升模型的精度，在项目中采用了一系列模型优化的方式，具体请参考[模型优化文档](./accuracy_improvement.md)
* 速度优化
 在完成模型精度优化之后，从以下两个方面对模型进行加速：

#### (1) 减少FPN部分的通道数量

将FPN部分的通道数量由原本的256减少至64，使用方式在定义模型[FasterRCNN](https://paddlex.readthedocs.io/zh_CN/develop/apis/models/detection.html#paddlex-det-fasterrcnn)类时设置参数`fpn_num_channels`为64即可，需要重新对模型进行训练。

#### (2) 减少测试阶段的候选框数量

将测试阶段RPN部分做非极大值抑制计算的候选框数量由原本的6000减少至500，将RPN部分做完非极大值抑制后保留的候选框数量由原本的1000减少至300。使用方式在定义模型[FasterRCNN](https://paddlex.readthedocs.io/zh_CN/develop/apis/models/detection.html#paddlex-det-fasterrcnn)类时设置参数`test_pre_nms_top_n`为500，`test_post_nms_top_n`为300。

采用Fluid C++预测引擎在Tesla P40上测试模型的推理时间（输入数据拷贝至GPU的时间、计算时间、数据拷贝至CPU的时间），输入大小设置为800x1333，加速前后推理时间如下表所示：

| 模型 | 推理时间 （ms/image）| VOC mAP (%) |
| -- | -- | -- |
| baseline | 66.51 | 88.87 |
| + fpn channel=64 + test proposal=pre/post topk 500/300 | 46.08 | 87.72 |



### 6 模型预测

运行如下代码：
``` bash
python code/infer.py
```

则可生成result.txt文件并显示预测结果图片，result.txt文件中会显示图片中每个检测框的位置、类别及置信度，并给出检测框的总个数，从而实现了钢筋自动计数。

预测结果如下：
<div align="center">
<img src="./images/predict.jpg"  width = "1000" />              </div>

### 7 模型导出
模型训练后保存在output文件夹，如果要使用PaddleInference进行部署需要导出成静态图的模型,运行如下命令，会自动在output文件夹下创建一个`inference_model`的文件夹，用来存放导出后的模型。

``` bash
paddlex --export_inference --model_dir=output/yolov3_darknet53/best_model --save_dir=output/inference_model --fixed_input_shape=608,608
```
**注意**：设定 fixed_input_shape 的数值需与 eval_transforms 中设置的 target_size 数值上保持一致。

### 8 模型上线选择
本案例面向GPU端的最终方案是选择二阶段检测模型FasterRCNN，其骨干网络选择加入了可变形卷积（DCN）的ResNet50_vd，训练时使用SSLD蒸馏方案训练得到的ResNet50_vd预训练模型，FPN部分的通道数量设置为64，训练阶段数据增强策略采用RandomHorizontalFlip、RandomDistort、RandomCrop，并加入背景图片，测试阶段的RPN部分做非极大值抑制计算的候选框数量由原本的6000减少至500、做完非极大值抑制后保留的候选框数量由原本的1000减少至300。

在Tesla P40的Linux系统下，对于输入大小是800 x 1333的模型，图像预处理时长为30ms/image，模型的推理时间为46.08ms/image，包括输入数据拷贝至GPU的时间、计算时间、数据拷贝至CPU的时间。

| 模型 | VOC mAP (%) | 推理时间 (ms/image)
| -- | -- | -- |
| FasterRCNN-ResNet50_vd_ssld | 81.05 | 48.62 |
| + dcn | 88.09 | 66.51 |
| + RandomHorizontalFlip/RandomDistort/RandomCrop | 90.23| 66.51 |
| + background images | 88.87 | 66.51 |
| + fpn channel=64 | 87.79 | 48.65 |
| + test proposal=pre/post topk 500/300 | 87.72 | 46.08 |

### 9 模型部署方式
模型部署采用了PaddleX提供的C++ inference部署方案，在改方案中提供了C#部署[Demo](https://github.com/PaddlePaddle/PaddleX/tree/develop/examples/C%23_deploy)，用户可根据实际情况自行参考。

<div align="center">
<img src="./images/Csharp.png"  width = "1000" />          
    </div>
