# 1.驱动、cuda安装
## 1.1安装驱动

```
sudo apt update
ubuntu-drivers devices
sudo ubuntu-drivers autoinstall
sudo reboot
nvidia-smi
```

# 2.Docker Engine
## 2.1Docker Engine介绍
在 ROS2 开发中的具体作用：
- 环境一致性：确保 ROS2 应用在开发、测试和生产环境中运行一致
- 依赖隔离：将 ROS2 应用及其依赖（特定库、工具）打包在独立容器中
- 快速部署：简化 ROS2 应用在多种平台（本地、云、边缘设备）的部署过程

实际效果：
- 开发效率：避免“在我机器上能运行”的问题，简化团队协作
- 资源利用：比虚拟机更轻量，共享主机内核，减少资源开销
- 版本管理：轻松管理不同 ROS2 版本（如同时使用 Humble 和 Foxy）

主要用途示例：
- 跨平台 ROS2 应用部署：在 x86 和 ARM 设备上运行相同容器
- 持续集成/测试：自动构建和测试 ROS2 节点
- 生产环境部署：在机器人或服务器上部署稳定的 ROS2 应用

# 3.cuda安装
## 3.1安装 CUDA Toolkit 11.8

按 NVIDIA 官方“cuda-keyring + apt”方式装，并且**只装 toolkit，不装 driver**
```
# 1) 加 NVIDIA CUDA 仓库 keyring（官方方式）
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.0-1_all.deb
sudo dpkg -i cuda-keyring_1.0-1_all.deb
sudo apt update

# 2) 只安装 Toolkit（不包含驱动）
sudo apt install -y cuda-toolkit-11-8

```

配置环境变量:
```
echo 'export CUDA_HOME=/usr/local/cuda-11.8' >> ~/.bashrc
echo 'export PATH=$CUDA_HOME/bin:$PATH' >> ~/.bashrc
(让 `nvcc` 等 CUDA 工具可以直接在终端调用)

echo 'export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
是让运行时找得到 CUDA 的动态库
source ~/.bashrc
```

- `~/.bashrc`：每次打开新的交互式终端都会执行它，所以把 export 写进去后，**永久生效**
- `source ~/.bashrc`：立即让当前这个终端会话也生效

验证:
```
which nvcc
nvcc --version
```

# 4.安装mujuco
## 4.1装编译工具 + OpenGL 相关依赖
```
sudo apt update
sudo apt install -y \
  git build-essential cmake pkg-config ninja-build \
  libgl1-mesa-glx libglfw3 libglew2.0 libosmesa6 \
  libegl1-mesa-dev

```
## 4.2安装venv 支持（可虚拟环境可不虚拟环境）

### 4.2.1安装venv虚拟环境 依赖
```
sudo apt update
sudo apt install -y python3.10-venv python3-pip

```
### 4.2.2清理失败的残留目录
```
rm -rf ~/venvs/mujoco
```

### 4.2.3重新创建 venv
```
python3.10 -m venv ~/venvs/mujoco
```
#### 4.2.4激活
```
source ~/venvs/mujoco/bin/activate
```

#### 4.2.5退出虚拟环境
```
deactivate
```

## 4.3命令及验证
### 4.3.1命令
```
python3 -m venv ~/venvs/mujoco
source ~/venvs/mujoco/bin/activate
pip install -U pip
pip install mujoco
python -c "import mujoco; print('mujoco ok', mujoco.__version__)"
```
### 4.3.2验证
```
python -m mujoco.viewer
```

### 4.3.3运行小模型

创建 `/tmp/pendulum.xml`
```
cat > /tmp/pendulum.xml <<'EOF'
<mujoco model="pendulum">
  <option timestep="0.01"/>
  <worldbody>
    <light pos="0 0 2"/>
    <body name="pole" pos="0 0 1">
      <joint name="hinge" type="hinge" axis="0 1 0" pos="0 0 0"/>
      <geom type="capsule" fromto="0 0 0 0 0 1" size="0.05" density="300"/>
    </body>
  </worldbody>
</mujoco>
EOF
```

创建`run_mujoco_viewer.py`:
```
cat > /tmp/run_mujoco_viewer.py <<'EOF'
import time
import mujoco
import mujoco.viewer

xml_path = "/tmp/pendulum.xml"
model = mujoco.MjModel.from_xml_path(xml_path)
data = mujoco.MjData(model)

data.qpos[0] = 1.0          # 给关节一个初始角度（弧度）
mujoco.mj_forward(model, data)

with mujoco.viewer.launch_passive(model, data) as viewer:
  while viewer.is_running():
    mujoco.mj_step(model, data)
    viewer.sync()
    time.sleep(model.opt.timestep)
EOF
```

运行：
```
python /tmp/run_mujoco_viewer.py
```

# 5.复现GraspNet Baseline
## 5.1安装miniconda
```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```
## 5.2创建虚拟环境
```
conda create -n graspnet python=3.10 -y
conda activate graspnet
python -V
```

错误：
”EnvironmentNameNotFound: Could not find conda environment: graspnet You can list all discoverable environments with conda info --envs.“

错误修复：
```
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

重新创建并激活环境：
```
conda create -n graspnet python=3.10 -y
conda activate graspnet

# 关键：禁止读取 ~/.local，防止 cv2/numpy 混入导致 ABI 崩溃
conda env config vars set PYTHONNOUSERSITE=1
conda deactivate
conda activate graspnet

python -c "import site; print('ENABLE_USER_SITE =', site.ENABLE_USER_SITE)"
# 期望输出：False
```

删除命令：
```
conda env remove -n graspnet -y
```
## 5.3安装 graspnet依赖
### 5.3.1conda 安装
```
conda install -y pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia
```
验证：
```
python -c "import torch; print('torch', torch.__version__); print('cuda available:', torch.cuda.is_available()); print('torch cuda:', torch.version.cuda)"
```

### 5.3.2升级pip 安装
```
python -m pip install -U pip setuptools wheel
```

### 5.3.3安装关键依赖
```

# transforms3d pin + 机器学习依赖
python -m pip install --no-cache-dir "transforms3d==0.3.1" scikit-learn

# 其他常用依赖（demo.py 用到）
python -m pip install --no-cache-dir open3d scipy pillow

```

transforms3d 不再炸：
```
python -c "import numpy as np; import transforms3d; print('numpy', np.__version__, 'transforms3d ok')"

```
## 5.4安装graspnetAPI
### 5.4.1下载graspAPI
```
mkdir -p ~/src
cd ~/src
git clone https://github.com/graspnet/graspnetAPI.git 
```
### 5.4.2下载graspAPI依赖
```
conda activate graspnet
python -m pip install -U pip setuptools wheel

cd ~graspnetAPI
python -m pip install -e .
# 后装 numpy pin
python -m pip install --no-cache-dir --force-reinstall "numpy==1.23.4"

```

出现错误：
```
    from graspnetAPI import GraspGroup
ModuleNotFoundError: No module named 'graspnetAPI'
```
解决：
```
cd ~/graspnet-baseline/graspnetAPI
echo "from .graspnetAPI import *" > __init__.py
```

验证：
```
python -c "import graspnetAPI; print('graspnetAPI ok', graspnetAPI.__file__)"
python -c "from graspnetAPI.grasp import GraspGroup; print('GraspGroup OK')"
```
### 5.4.3处理 `pip check` 常见告警
```
python -m pip check
```
## 5.5安装编译GraspNet Baseline
### 5.5.1安装编译
```
git clone https://github.com/graspnet/graspnet-baseline.git
cd graspnet-baseline


# 编译 pointnet2
cd pointnet2
python setup.py install
cd ..

# 编译 knn
cd knn
python setup.py install
cd ..
```

### 5.4.2准备 graspnet-baseline 代码并修复 torch._six
进入 baseline：
```
cd ~/graspnet-baseline
```

修改代码dataset/graspnet_dataset.py：
```
from torch._six import container_abcs
#替换成兼容写法：
try:
    from torch._six import container_abcs  # torch < 2.0
except Exception:
    import collections.abc as container_abcs  # torch >= 2.0

```

验证 dataset import：
```
python -c "from dataset.graspnet_dataset import GraspNetDataset; print('dataset import ok')"

```
## 5.5运行官方示例demo.py
### 5.1手动构建文档

在ubuntu22.04系统上直接安装：
```
sudo apt update
sudo apt install -y latexmk texlive-latex-base texlive-latex-recommended texlive-latex-extra texlive-fonts-recommended
```
### 5.2下载label和weight
#### 5.2.1下载
```
https://pan.baidu.com/s/1HN29P-csHavJF-R_wec6SQ?_at_=1712722990153
```

#### 5.2.2将tolerance.tar移动到dataset目录下面
```
cd dataset
tar -xvf tolerance.tar
```

#### 5.2.3下载预训练权重

checkpoint-rs.tar和checkpoint-kn.tar是分别使用 RealSense 数据和 Kinect 数据进行训练

[checkpoint-rs.tar]  https://pan.baidu.com/s/1Eme60l39tTZrilF0I86R5A
[checkpoint-kn.tar]  https://pan.baidu.com/s/1QpYzzyID-aG5CgHjPFNB9g

#### 5.2.4正确放置
```
cd ~/graspnet-baseline
mkdir -p logs/log_kn
# 把 checkpoint-kn.tar 放进 log_kn 并改名为 checkpoint.tar
mv -f checkpoint-kn.tar logs/log_kn/checkpoint.tar
# 看看文件是否在位
ls -lh logs/log_kn/checkpoint.tar
```
#### 5.2.5检测文件依赖
```
conda activate graspnet
cd ~/graspnet-baseline

# 必须存在这些
ls -lah demo.py command_demo.sh
ls -lah doc/example_data
ls -lah logs/log_kn/checkpoint.tar

#确认 PyTorch 能用 GPU
python -c "import torch; print('torch', torch.__version__); print('cuda avail', torch.cuda.is_available()); print('cuda', torch.version.cuda)"
#`cuda avail` 期望是 `True`

#安装 baseline 运行所需 Python 依赖
python -c "from PIL import Image; import scipy.io as scio; print('PIL+scipy.io ok')"

#设置 GPU 架构（Quadro RTX 4000 = 7.5）
export TORCH_CUDA_ARCH_LIST="7.5"

#检查你有没有 nvcc
nvcc --version
```

### 5.3指定 GPU 架构（例如 RTX 4000 -> 7.5）
```
export TORCH_CUDA_ARCH_LIST="7.5"
```

### 5.4运行demo.py
在仓库根目录：
```
cd ~/graspnet-baseline
python demo.py --checkpoint_path logs/log_kn/checkpoint.tar

```

或者修改demo.py文件：
```
parser.add_argument('--checkpoint_path', required=True, help='Model checkpoint path')
```

修改成：
```
parser.add_argument('--checkpoint_path',
                        default='/home/robot/graspnet-baseline/logs/log_kn/checkpoint.tar',  # 你的路径
                        help='Model checkpoint path')
```

# 6.vscode无法启动rviz2
## 6.1不行就操作6.2
```
source /opt/ros/humble/setup.bash
source install/setup.bash
```

## 6.2把 snap 的 VSCode 卸掉，改装 deb/apt 版
```
sudo snap remove code
sudo apt update
sudo apt install -y wget gpg
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg >/dev/null
echo "deb [arch=amd64] https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources.list.d/vscode.list
sudo apt update
sudo apt install -y code

```

# 7.复现graspnet-mujuco抓取
## 7.1装系统依赖
```
sudo apt update
sudo apt install -y git cmake build-essential python3-dev python3-venv \
  libgl1-mesa-dev libglib2.0-0 libegl1-mesa libxrender1 libsm6 libxext6 \
  libeigen3-dev
```

## 7.2安装graspnet-baseline

## 7.3运行main.py错误解决
### 7.3.1
错误：
```
    import spatialmath as sm
ModuleNotFoundError: No module named 'spatialmath'
```

解决：
```
pip install spatialmath-python
```

### 7.3.2

运行main.py出现错误：
```
    from graspnetAPI import GraspGroup
ModuleNotFoundError: No module named 'graspnetAPI'
```

解决：
```
cd ~/manipulator_grasp/graspnet-baseline
pip uninstall -y graspnetAPI
cd graspnetAPI
pip install -e . --no-deps --no-build-isolation
```

### 7.3.3
运行main.py出现错误：
```
import roboticstoolbox as rtb
ModuleNotFoundError: No module named 'roboticstoolbox'
```

解决：
```
python -m pip install -U pip
python -m pip install roboticstoolbox-python spatialmath-python
```

装完验证:
```
python -c "import roboticstoolbox as rtb; import spatialmath as sm; print('rtb ok')"
```

### 7.3.4
运行main.py出现错误：
```
    import modern_robotics as mr
ModuleNotFoundError: No module named 'modern_robotics'
```

解决：
```
conda activate graspnet
python -m pip install -U modern_robotics
```

验证：
```
python -c "import modern_robotics as mr; print('modern_robotics OK')"
```

### 7.3.5
运行main.py出现错误：
```
    import mujoco
ModuleNotFoundError: No module named 'mujoco'
```

解决：
```
python -m pip install -U mujoco
```

验证：
```
python -c "import mujoco; print('mujoco ok', mujoco.__version__)"
```

### 7.3.6
运行main.py出现错误：
```
FileNotFoundError: [Errno 2] No such file or directory: './logs/log_rs/checkpoint-rs.tar'
```

解决：
在main.py文件修改代码：
```
checkpoint_path = './logs/log_rs/checkpoint-rs.tar'
```

修改为：
```
checkpoint_path = './logs/log_rs/checkpoint.tar'
```

### 7.3.7弹出Open3d和mujuco窗口后不动

解决：
关闭Open3d可视化窗口