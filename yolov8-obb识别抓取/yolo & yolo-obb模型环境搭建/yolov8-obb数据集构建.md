```
conda activate yolov8
pip install x-anylabeling-cvhub[gpu-cu11]
git clone https://github.com/CVHub520/X-AnyLabeling.git cd X-AnyLabeling
cd X-AnyLabeling
pip install -e .[gpu-cu11]
xanylabeling
```

### 1.数据集准备
#### 1.下载X-AnyLabeling自动化标注软件
![[Pasted image 20251031174821.png]]
选择“mobileSAM-(vit-huge)”模型辅助化标注
#### 2.导出obb格式
在需要导出的文件夹创建分类标签“classes”txt格式文件
![[Pasted image 20251031175239.png]]

在X-AnyLabeling软件页面选择“旋转框格式”
![[Pasted image 20251031175104.png]]

### 2.模型训练
#### 1.迁移数据集
C:/datasets/box-pen-obb路径创建/box-pen-obb文件夹
#### 2.创建data.yaml
C:/datasets/box-pen-obb路径创建data.yaml：
```
# Box-Pen OBB Dataset Configuration
train: images/train  # 相对path
val: images/val
# Classes
names:
  0: pen
  1: box
  
# 可选：数据增强
augment: True
degrees: 10.0      # 旋转增强（-10到+10度）
translate: 0.1     # 平移
scale: 0.5         # 缩放
shear: 0.0         # 剪切
perspective: 0.0   # 透视
flipud: 0.0        # 垂直翻转
fliplr: 0.5        # 水平翻转50%概率
mosaic: 1.0        # Mosaic增强
mixup: 0.0         # Mixup
copy_paste: 0.0
```

#### 3.创建训练脚本train-obb.py
```
# train_obb.py
from ultralytics import YOLO
import torch

# 设置设备
device = 'cuda' if torch.cuda.is_available() else 'cpu'
print(f"使用设备: {device}")

# 加载预训练模型（首次自动下载）
model = YOLO('yolov8n-obb.pt')  # nano版，轻量快速
# 可选：yolov8s-obb.pt (小), yolov8m-obb.pt (中), yolov8l-obb.pt (大)

# 训练参数配置
results = model.train(
    data='C:/Users/JasonLee/OneDrive/Desktop/ultralytics-main/datasets/box-pen-obb/data.yaml',  # 数据集配置
    epochs=100,              # 训练轮数（建议50-200）
    imgsz=640,               # 输入尺寸（640/1024，更大更精确）
    batch=16,                # 批次大小（根据显存调整，16/32/64）
    device=device,           # GPU或CPU
    project='runs/obb',      # 保存路径
    name='box-pen-exp1',     # 实验名称
    exist_ok=True,           # 允许覆盖
    
    # 优化器设置
    optimizer='AdamW',       # AdamW/SGD（OBB推荐AdamW）
    lr0=0.01,                # 初始学习率
    lrf=0.01,                # 最终学习率（lr0 * lrf）
    momentum=0.937,          # SGD动量
    weight_decay=0.0005,     # 权重衰减
    
    # 训练策略
    patience=50,             # 早停轮数（无提升则停）
    save=True,               # 保存检查点
    save_period=10,          # 每10轮保存一次
    cache=False,             # 缓存图像（RAM充足设True）
    workers=0,               
    
    # 数据增强（覆盖yaml设置）
    hsv_h=0.015,             # 色调增强
    hsv_s=0.7,               # 饱和度
    hsv_v=0.4,               # 亮度
    degrees=10.0,            # 旋转±10度（OBB关键）
    translate=0.1,           # 平移10%
    scale=0.5,               # 缩放50%
    shear=0.0,               # 剪切
    perspective=0.0,         # 透视
    flipud=0.0,              # 垂直翻转
    fliplr=0.5,              # 水平翻转
    mosaic=1.0,              # Mosaic增强（OBB有效）
    mixup=0.0,               # Mixup（OBB慎用）
    copy_paste=0.0,          # Copy-Paste

    # 验证设置
    val=True,                # 训练时验证
    plots=True,              # 生成图表
    verbose=True             # 详细输出

)
# 训练完成后
print(f"最佳权重: {model.trainer.best}")
print(f"最终权重: {model.trainer.last}")
```
#### 4.模型预测

yolo detect predict model=runs/obb/box-pen-exp1/weights/best.pt source=./datasets/box-pen-video.mp4 show=True


yolo detect predict model=runs/obb/box-pen-exp1/weights/best.pt source=./datasets/pen-box-video1.mp4 show=True