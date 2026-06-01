```
# train_obb.py

from ultralytics import YOLO

import torch

  

# 设置设备

device = 'cuda' if torch.cuda.is_available() else 'cpu'

print(f"使用设备: {device}")

  

# 加载预训练模型（首次自动下载）

# model = YOLO('yolov8n-obb.pt')  # nano版，轻量快速

# model =  YOLO('C:/Users/JasonLee/OneDrive/Desktop/ultralytics-main/runs/obb/box-pen-cube-finetune1/weights/best.pt')

model =  YOLO('C:/Users/JasonLee/OneDrive/Desktop/ultralytics-main/yolov8n-obb.pt')

# 可选：yolov8s-obb.pt (小), yolov8m-obb.pt (中), yolov8l-obb.pt (大)

  

# 训练参数配置

results = model.train(

    data='C:/Users/JasonLee/OneDrive/Desktop/ultralytics-main/datasets/box-pen-obb/data.yaml',  # 数据集配置

    # epochs=100,              # 训练轮数（建议50-200）

    epochs=60,

    imgsz=640,               # 输入尺寸（640/1024，更大更精确）

    batch=16,                # 批次大小（根据显存调整，16/32/64）

    device=device,           # GPU或CPU

    project='runs/obb',      # 保存路径

    # name='box-pen-exp1',     # 实验名称

    name='box-pen-cube-finetune1',     # 实验名称

    exist_ok=True,           # 允许覆盖

    # 优化器设置

    optimizer='AdamW',       # AdamW/SGD（OBB推荐AdamW）

    # lr0=0.01,                # 初始学习率

    lr0=0.001,                # 初始学习率

    lrf=0.01,                # 最终学习率（lr0 * lrf）

    momentum=0.937,          # SGD动量

    weight_decay=0.0005,     # 权重衰减

    # 训练策略

    # patience=50,             # 早停轮数（无提升则停）

    patience=20,           # ✅ 早停更快

    save=True,               # 保存检查点

    save_period=10,          # 每10轮保存一次

    cache=False,             # 缓存图像（RAM充足设True）

    workers=0,               # 数据加载线程（8核CPU用4-8）

    # 数据增强（覆盖yaml设置）

    hsv_h=0.015,             # 色调增强

    hsv_s=0.7,               # 饱和度

    hsv_v=0.4,               # 亮度

    degrees=10.0,            # 旋转±10度（OBB关键）

    translate=0.1,           # 平移10%

    scale=0.25,               # 缩放50%

    shear=0.0,               # 剪切

    perspective=0.0,         # 透视

    flipud=0.0,              # 垂直翻转

    fliplr=0.5,              # 水平翻转

    mosaic=0.3,              # Mosaic增强（OBB有效）

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