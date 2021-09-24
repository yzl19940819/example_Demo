# 钢筋计数安卓版demo


### 1 环境准备
paddlex==1.3.11

paddlelite==2.6.1

模型说明：PaddleX提供了丰富的视觉模型，由于本项目最终需要部署在安卓手机上运行，考虑到速度已经内存资源占用等问题，本项目中采用yoloV3作为检测模型，然后使用轻量级的MobileNetV1
作为backbone进行钢筋计数。参考本demo教程进行训练，然后导出为静态图模型。


### 2 静态图转换为paddle lite文件
由于我们的模型最终要在手机上进行部署，因此，考虑到手机上资源的有限性以及推理的速度问题，我们需要采用飞桨的移动端框架PaddleLite来完成模型的转化和部署。

首先安装PaddleLite：
``` shell
pip install paddlelite==2.6.1
```

接下来进行模型转换，详细代码如下：
```python
# 引用Paddlelite预测库
from paddlelite.lite import *

# 1. 创建opt实例
opt=Opt()
# 2. 指定输入模型地址 
opt.set_model_file('./output/inference_model/__model__')
opt.set_param_file('./output/inference_model/__params__')
# 3. 指定转化类型： arm、x86、opencl、npu
opt.set_valid_places("arm")
# 4. 指定模型转化类型： naive_buffer、protobuf
opt.set_model_type("naive_buffer")
# 5. 输出模型地址
opt.set_optimize_out("./output/model")
# 6. 执行模型优化
opt.run()
```

运行上述python代码，即可将静态图模型转换为paddle lite对应的nb文件，这个文件可以在安卓上进行部署。

### 3 安卓demo

PaddleX官网提供了详细的安卓[部署教程](https://paddlex.readthedocs.io/zh_CN/release-1.3/deploy/paddlelite/android.html)。

如果用户训练好了自己的钢筋检测模型，想要替换部署，那么只需要按照前面的步骤准备好Lite模型(.nb文件)和yml配置文件，然后在Android Studio的project视图中进行替换操作。
**替换操作：**
* **替换nb文件** 将.nb文件拷贝到/src/main/assets/model/目录下, 根据.nb文件的名字，修改文件/src/main/res/values/strings.xml中的MODEL_PATH_DEFAULT；
* **替换yml文件** 将.yml文件拷贝到/src/main/assets/config/目录下，根据.yml文件的名字，修改文件/src/main/res/values/strings.xml中的YAML_PATH_DEFAULT；
* **替换测试图片** 可根据需要替换测试图片，将图片拷贝到/src/main/assets/images/目录下，根据图片文件的名字，修改文件/src/main/res/values/strings.xml中的IMAGE_PATH_DEFAULT；


将工程导入后，点击菜单栏的Run->Run ‘App’按钮，在弹出的”Select Deployment Target”窗口选择已经连接的Android设备，然后点击”OK”按钮。

项目效果如下图所示，默认打开app后自动加载模型，点击predict即可执行预测，预测结果在底部显示（输出钢筋计数结果和推理时间）。

<div align="center">
<img src="./images/phone_pic.jpg"  width = "1000" />              </div>
