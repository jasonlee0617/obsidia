### 1.公共数据集的构建
#### 1.网上搜集公开数据集”box“ "pen"
#### 2.创建”box-pen“数据集
##### images:存放图片
train:训练集图片
val:验证集图片
test:测试集图片
##### labels:存放标签
train:训练集标签文件，要与训练集图片名称一一对应
val:验证集图片，要与验证集图片名称一一对应
test:测试集图片，要与验证集图片名称一一对应

#### 3.放进ultralytics-main根目录下的”datasets“

### 2.无迁移学习模型训练
#### 1.根目录下创建”box-pen-train(no transfer_learning)“文件夹

#### 2.创建”no-transfer-learning-train.py“文件

```
from ultralytics import YOLO
import yaml
import os

def create_enhanced_config():
    """创建增强的训练配置"""
    # 检查原始配置
    original_config = "box-pen-train/yolo-box-pen.yaml"
    if not os.path.exists(original_config):
        print(f"错误：找不到原始配置文件 {original_config}")
        return None

    # 读取原始配置
    with open(original_config, 'r', encoding='utf-8') as f:
        config = yaml.safe_load(f)

    # 添加强化的数据增强参数
    enhanced_config = {
        **config,  # 保持原有配置
        # 强化数据增强参数（防过拟合的关键）
        'hsv_h': 0.03,        # HSV色调变化 (0-1)
        'hsv_s': 0.8,         # HSV饱和度变化 (0-1)
        'hsv_v': 0.5,         # HSV明度变化 (0-1)
        'degrees': 30.0,      # 旋转角度 (-degrees to +degrees)
        'translate': 0.2,     # 平移 (0-1)
        'scale': 0.8,         # 缩放 (0-1)
        'shear': 5.0,         # 剪切角度 (-shear to +shear)
        'perspective': 0.0003,# 透视变换 (0-0.001)
        'flipud': 0.5,        # 垂直翻转概率 (0-1)
        'fliplr': 0.5,        # 水平翻转概率 (0-1)
        'mosaic': 1.0,        # 马赛克增强概率 (0-1)
        'mixup': 0.3,         # 图像混合概率 (0-1)
        'copy_paste': 0.2,    # 复制粘贴增强 (0-1)
    }

    # 保存增强配置
    enhanced_config_path = "box-pen-train(no transfer_learning)/yolo-box-pen-enhanced.yaml"
    with open(enhanced_config_path, 'w', encoding='utf-8') as f:
        yaml.dump(enhanced_config, f, default_flow_style=False)
    print(f"增强配置已保存: {enhanced_config_path}")
    return enhanced_config_path
    
def retrain_with_strong_augmentation():
    """使用强数据增强重新训练"""
    print("=== 使用强数据增强重新训练模型 ===")
    # 创建增强配置
    config_path = create_enhanced_config()
    if not config_path:
        return

    model_path = "yolov8n.pt"
    print("从预训练模型开始...")
    # 加载模型
    model = YOLO(model_path)
    # 训练参数（防过拟合）
    training_args = {
        'data': config_path,
        'epochs': 80,              # 适当减少epochs
        'batch': 16,
        'imgsz': 640,
        'workers': 0,
        
        # 防过拟合参数
        'lr0': 0.01,              # 初始学习率
        'lrf': 0.01,              # 最终学习率比例
        'momentum': 0.937,
        'weight_decay': 0.0005,   # L2正则化
        'warmup_epochs': 5,       # 学习率预热
        'warmup_momentum': 0.8,
        'warmup_bias_lr': 0.1,

        # 验证和保存
        'patience': 25,           # 早停耐心值
        'save_period': 10,        # 保存间隔
        'val': True,
        'plots': True,
        'save': True,

        # 其他优化
        'cos_lr': True,           # 余弦学习率调度
        'close_mosaic': 15,       # 最后15个epoch关闭mosaic
        'amp': True,              # 自动混合精度
        'cache': False,           # 不缓存图像（节省内存）
    }

    print("\n开始重新训练...")
    print("训练配置:")
    for key, value in training_args.items():
        if key in ['data']:
            continue
        print(f"  {key}: {value}")
    try:

        # 开始训练
        results = model.train(**training_args)
        return True

    except Exception as e:
    
        print(f"训练失败: {e}")
        return False

  

if __name__ == "__main__":

    retrain_with_strong_augmentation()
```

#### 3.运行”no-transfer-learning-train.py“文件

#### 4.生成”train-no-transfer_learning“模型

==train-no-transfer_learning模型存在过拟合，因此需要进行迁移模型，即根据真实场景下的图片进行迁移学习，学习真实场景下的物体特征==

### 3.迁移学习模型训练
#### 1.在根目录下创建”box-pen-transfer_learning“文件夹

#### 2.创建”transfer_learning.py“文件

```
import os
import yaml
from ultralytics import YOLO
from pathlib import Path

def create_target_config():

    """创建目标域数据配置"""
    config = {
        'path': 'datasets/target_domain_data',
        'train': 'images/train',
        'val': 'images/train',  # 数据少时训练和验证用同一批
        'names': {0: 'pen', 1: 'box'},

        # 适合微调的温和数据增强
        'hsv_h': 0.01,
        'hsv_s': 0.3,
        'hsv_v': 0.2,
        'degrees': 10.0,
        'translate': 0.05,
        'scale': 0.95,
        'shear': 2.0,
        'flipud': 0.1,
        'fliplr': 0.5,
        'mosaic': 0.1,
        'mixup': 0.0,

    }

    with open('target_domain_config.yaml', 'w') as f:
        yaml.dump(config, f, default_flow_style=False)
        
    return 'target_domain_config.yaml'

def fine_tune_model():

    """执行微调训练"""
    print("开始迁移学习微调...")
    
    # 选择基础模型
    base_models = [
        "runs/detect/train-no-transfer_learning/weights/best.pt",
    ]

    selected_model = None

    for model_path in base_models:
        if os.path.exists(model_path):
            selected_model = model_path
            print(f"使用基础模型: {model_path}")
            break

    if not selected_model:
        print("未找到基础模型，请检查路径")
        return

    # 创建配置
    config_path = create_target_config()
    
    # 加载模型
    model = YOLO(selected_model)

    # 微调参数
    train_params = {
        'data': config_path,
        'epochs': 20,              # 适当减少epochs
        'batch': 16,
        'imgsz': 640,
        'workers': 0,

        # 防过拟合参数
        'lr0': 0.01,              # 初始学习率
        'lrf': 0.01,              # 最终学习率比例
        'momentum': 0.937,
        'weight_decay': 0.0005,   # L2正则化
        'warmup_epochs': 5,       # 学习率预热
        'warmup_momentum': 0.8,
        'warmup_bias_lr': 0.1,

        # 验证和保存
        'patience': 25,           # 早停耐心值
        'save_period': 10,        # 保存间隔
        'val': True,
        'plots': True,
        'save': True,

        # 其他优化
        'cos_lr': True,           # 余弦学习率调度
        'close_mosaic': 15,       # 最后15个epoch关闭mosaic
        'amp': True,              # 自动混合精度
        'cache': False,           # 不缓存图像（节省内存）

    }

    print("开始训练...")
    results = model.train(**train_params)

if __name__ == "__main__":
    fine_tune_model()
```

#### 3.运行”transfer_learning.py“文件

#### 4.生成”transfer_learning“模型

**”transfer_learning“模型问题：**
1.**遮挡检测失败**：当pen在box内部时无法检测到
2.**部分遮挡检测不准确**：只能检测到露在box外面的pen部分

**改进方法：**
**“target_domain_data”训练集中增加”pen在box内部“的照片**

#### 5.在”transfer_learning“模型基础上进行迁移训练

##### 1.修改”transfer_learning.py“部分代码
原代码：
```
    # 选择基础模型
    base_models = [
        "runs/detect/train-no-transfer_learning/weights/best.pt",
    ]
```

修改后代码：
```
    # 选择基础模型
    base_models = [
        "runs/detect/transfer_learning/weights/best.pt",
    ]
```

##### 2.“target_domain_data”训练集中增加”pen在box内部“的照片

##### 3.运行"transfer_learning.py"文件

##### 4.生成“transfer_learning_plus”模型
