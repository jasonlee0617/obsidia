- [[#“acadosConfig.cmake”编译报错|“acadosConfig.cmake”编译报错]]
	- [[#“acadosConfig.cmake”编译报错#1.acados环境路径写入|1.acados环境路径写入]]
	- [[#“acadosConfig.cmake”编译报错#2.给 acados 创建 `.venv`|2.给 acados 创建 `.venv`]]
	- [[#“acadosConfig.cmake”编译报错#3.安装 acados Python 接口|3.安装 acados Python 接口]]
	- [[#“acadosConfig.cmake”编译报错#4.再次用 `.venv/bin/python3` 验证|4.再次用 `.venv/bin/python3` 验证]]
	- [[#“acadosConfig.cmake”编译报错#5.清理编译|5.清理编译]]
- [[#1. 安装系统基础依赖|1. 安装系统基础依赖]]
- [[#2.安装 Rust 环境|2.安装 Rust 环境]]
- [[#3. 安装 Node.js 与 pnpm|3. 安装 Node.js 与 pnpm]]
- [[#4. 克隆项目并安装前端依赖|4. 克隆项目并安装前端依赖]]
- [[#5. 解决编译报错（核心修正步骤）|5. 解决编译报错（核心修正步骤）]]
- [[#6. source环境出现警告错误|6. source环境出现警告错误]]
- [[#7.验证官方 Demo|7.验证官方 Demo]]
- [[#8.用本地模型替换 sample 模型|8.用本地模型替换 sample 模型]]
- [[#1.工作模式|1.工作模式]]
- [[#2.在 Docker 里只编译 WVCSC 工作区.|2.在 Docker 里只编译 WVCSC 工作区.]]
- [[#3.容器重新验证官方 Demo|3.容器重新验证官方 Demo]]
- [[#4.用本地模型替换 sample 模型|4.用本地模型替换 sample 模型]]

# 1.源码编译
```
# 安装 CycloneDDS
sudo apt install -y ros-humble-rmw-cyclonedds-cpp
#写入环境变量：
echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
echo "export ROS_DOMAIN_ID=0" >> ~/.bashrc
source ~/.bashrc

# 1. 进入 home
cd /home/robot

# 2. 克隆
git clone https://github.com/autowarefoundation/autoware.git
cd autoware

# 3. 切稳定版本
git checkout 1.8.0

# 4. 安装 Ansible 和依赖
bash ansible/scripts/install-ansible.sh
ansible-galaxy collection install -f -r ansible-galaxy-requirements.yaml

# 有 NVIDIA GPU
ansible-playbook autoware.dev_env.install_dev_env

# 没有 NVIDIA GPU
# ansible-playbook autoware.dev_env.install_dev_env --skip-tags nvidia

# 5. 导入源码
sudo apt update sudo apt install python3-vcstool
mkdir -p src
vcs import src < repositories/autoware.repos

# 6. rosdep
source /opt/ros/humble/setup.bash
rosdep update
rosdep install -y --from-paths src --ignore-src --rosdistro $ROS_DISTRO

# 7. 编译
colcon build \
  --symlink-install \
  --parallel-workers 2 \
  --cmake-args -DCMAKE_BUILD_TYPE=Release

# 8. source
source install/setup.bash

# 9. 检查
ros2 pkg prefix autoware_launch

# 10. 测试 launch 参数
ros2 launch autoware_launch planning_simulator.launch.xml --show-args

## 11. 最后能运行：
ros2 launch autoware_launch planning_simulator.launch.xml \
  map_path:=$HOME/autoware_data/maps/sample-map-planning \
  vehicle_model:=sample_vehicle \
  sensor_model:=sample_sensor_kit
## RViz 中能看到地图、车辆，设置起点和终点后能切到 Auto 并运动。
```

##  “acadosConfig.cmake”编译报错

背景：acados已经安装，但是系统没有识别到

**acados 已经存在，问题不是没安装，而是 CMake 查找路径没有配置正确**

解决办法：
### 1.acados环境路径写入
```
#在bashrc中写入相关路径，根据你下载安装编译的acados路径而定
gedit ~/.bashrc
```

文件末尾空白处写入：
```
export ACADOS_INSTALL_DIR="/home/robot/桌面/acados_toolkit/acados"  
export ACADOS_SOURCE_DIR="$ACADOS_INSTALL_DIR"  
export acados_DIR="$ACADOS_INSTALL_DIR/cmake"  
export CMAKE_PREFIX_PATH="$ACADOS_INSTALL_DIR:$CMAKE_PREFIX_PATH"  
export LD_LIBRARY_PATH="$ACADOS_INSTALL_DIR/lib:$LD_LIBRARY_PATH"
```

### 2.给 acados 创建 `.venv`

```
#执行
cd /home/robot/桌面/acados_toolkit/acados
#安装 Python 虚拟环境工具：
sudo apt update
sudo apt install -y python3-venv python3-pip python3-dev
#创建 `.venv`：
python3 -m venv .venv
#检查：
ls /home/robot/桌面/acados_toolkit/acados/.venv/bin/python3
#现在应该能看到：
/home/robot/桌面/acados_toolkit/acados/.venv/bin/python3
```

### 3.安装 acados Python 接口

```
#进入 acados 目录：
cd /home/robot/桌面/acados_toolkit/acados
#激活虚拟环境,安装依赖
source .venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install "numpy<2" scipy matplotlib casadi==3.7.2
python -m pip install -e interfaces/acados_template
deactivate
```
### 4.再次用 `.venv/bin/python3` 验证

```
/home/robot/桌面/acados_toolkit/acados/.venv/bin/python3 - <<'EOF'
import sys
import casadi
import numpy
import acados_template

print("Python:", sys.executable)
print("casadi:", casadi.__version__)
print("numpy:", numpy.__version__)
print("acados_template OK")
EOF
```

正确结果应该类似：
```
Python: /home/robot/桌面/acados_toolkit/acados/.venv/bin/python3
casadi: 3.7.2
numpy: 1.xx.x
acados_template OK
```

### 5.清理编译

```
rm -rf src/*/*/autoware_path_optimizer/src/acados_mpc/c_generated_code
cd /home/robot/autoware

source /opt/ros/humble/setup.bash
source ~/.bashrc

export ACADOS_INSTALL_DIR="/home/robot/桌面/acados_toolkit/acados"
export ACADOS_SOURCE_DIR="$ACADOS_INSTALL_DIR"
export acados_DIR="$ACADOS_INSTALL_DIR/cmake"
export CMAKE_PREFIX_PATH="$ACADOS_INSTALL_DIR:$CMAKE_PREFIX_PATH"
export LD_LIBRARY_PATH="$ACADOS_INSTALL_DIR/lib:$LD_LIBRARY_PATH"

colcon build \
  --symlink-install \
  --packages-select autoware_path_optimizer \
  --cmake-args \
    -DCMAKE_BUILD_TYPE=Release \
    -Dacados_DIR="$ACADOS_SOURCE_DIR/cmake"
```

# 2.Autoware Build GUI可视化界面 编译安装全流程
## 1. 安装系统基础依赖
```
sudo apt install libwebkit2gtk-4.1-0 libjavascriptcoregtk-4.1-0 libsoup-3.0-0 libsoup-3.0-common
```

## 2.安装 Rust 环境
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs/ | sh
```

安装完成后，终端无法直接识别 `rustc` 命令。需要手动刷新当前 Shell 的环境变量：
```
source "$HOME/.cargo/env"
```

## 3. 安装 Node.js 与 pnpm
```
sudo apt update
sudo apt install nodejs
npm install -g pnpm
```

## 4. 克隆项目并安装前端依赖

拉取 Autoware Build GUI 的源码并进入目录：
```
git clone https://github.com/leo-drive/autoware-build-gui.git cd autoware-build-gui pnpm i
```

## 5. 解决编译报错（核心修正步骤）

**修正操作**：安装缺失的开发依赖库：
```
sudo apt install libatk1.0-dev libgdk-pixbuf2.0-dev libgtk-3-dev libwebkit2gtk-4.1-dev
```
_如果在后续编译中提示找不到 `libclang`，
可额外执行
```
sudo apt install libclang-dev
```

补全依赖后，强制重新安装 pnpm 依赖以确保环境干净，最后启动开发版应用：
```
pnpm install --force 
pnpm tauri dev
```

## 6. source环境出现警告错误

问题：
```
robot@eisa:~/autoware$ source install/setup.bash not found: "/home/robot/autoware/install/autoware_tensorrt_common/share/autoware_tensorrt_common/local_setup.bash" not found: "/home/robot/autoware/install/autoware_tensorrt_classifier/share/autoware_tensorrt_classifier/local_setup.bash" not found: "/home/robot/autoware/install/autoware_tensorrt_plugins/share/autoware_tensorrt_plugins/local_setup.bash" not found: "/home/robot/autoware/install/bevdet_vendor/share/bevdet_vendor/local_setup.bash"
```

解决办法：
```
cd /home/robot/autoware

for pkg in \
  autoware_tensorrt_common \
  autoware_tensorrt_classifier \
  autoware_tensorrt_plugins \
  bevdet_vendor
do
  mkdir -p install/$pkg/share/$pkg
  touch install/$pkg/share/$pkg/local_setup.bash
  touch install/$pkg/share/$pkg/local_setup.sh
  touch install/$pkg/share/$pkg/local_setup.zsh
done
```

然后重新 source：
```
source /opt/ros/humble/setup.bash
source /home/robot/autoware/install/setup.bash
```
## 7.验证官方 Demo

在容器里执行：
```
ros2 launch autoware_launch planning_simulator.launch.xml \
  map_path:=/home/robot/autoware_data/maps/sample-map-planning \
  vehicle_model:=sample_vehicle \
  sensor_model:=sample_sensor_kit
```

## 8.用本地模型替换 sample 模型

```
ros2 launch autoware_launch planning_simulator.launch.xml \
  map_path:=/home/robot/autoware_data/maps/sample-map-planning \
  vehicle_model:=wvcsc_vehicle \
  sensor_model:=wvcsc_sensor_kit
```

# 3.docker挂载形式

列出下载的docker:
```
docker images | grep autoware
```

autoware.universe docker下载参照[ROS 2 移动机器人导航与自动驾驶/Autoware学习文档]()

输出如下：
```
WARNING: This output is designed for human readability. For machine-readable output, please use --format.
ghcr.io/autowarefoundation/autoware:core-humble       dd915352ecbd       5.94GB          1.5GB        
ghcr.io/autowarefoundation/autoware:universe-humble   5c165246be6f       10.2GB          2.3GB   U    
robot@eisa:~/autoware
```

## 1.工作模式

```
Docker 里已有 Autoware
        ↓
挂载你的 WVCSC 工作区
        ↓
只编译 WVCSC_S2Z_UTB_ARM
        ↓
source WVCSC 的 install
        ↓
运行你的车身模型、传感器、vehicle_interface、bringup
```

宿主机目录建议保持：
```
/home/robot/WVCSC_S2Z_UTB_ARM          # 实车工作区
/home/robot/autoware_data              # 地图、rosbag、模型数据
/home/robot/autoware                   # Autoware 源码，只查看，不编译
/home/robot/autoware_scripts           # docker启动脚本
```

启动docker挂载：
```
~/autoware_scripts/run_autoware_universe.sh 
```
## 2.在 Docker 里只编译 WVCSC 工作区.

进入容器后：
```
cd /home/aw/WVCSC_S2Z_UTB_ARM
```

建议不要直接用默认 `build/install/log`，而是单独给 Docker 用一套目录，避免和你宿主机以前编译的结果混在一起：
```
colcon --log-base log_docker build \
  --symlink-install \
  --build-base build_docker \
  --install-base install_docker
```

编译完成后：
```
source install_docker/setup.bash
```

检查你的包：
```
ros2 pkg prefix wvcsc_vehicle_interface
ros2 pkg prefix wvcsc_vehicle_description
ros2 pkg prefix wvcsc_sensor_kit_description
ros2 pkg prefix wvcsc_autoware_bringup
```

## 3.容器重新验证官方 Demo

在容器里执行：
```
ros2 launch autoware_launch planning_simulator.launch.xml \
  map_path:=/home/aw/autoware_data/maps/sample-map-planning \
  vehicle_model:=sample_vehicle \
  sensor_model:=sample_sensor_kit
```

## 4.用本地模型替换 sample 模型

```
ros2 launch autoware_launch planning_simulator.launch.xml \
  map_path:=/home/aw/autoware_data/maps/sample-map-planning \
  vehicle_model:=wvcsc_vehicle \
  sensor_model:=wvcsc_sensor_kit
```

出现错误：
```
[INFO] [launch]: All log files can be found below /home/aw/.ros/log/2026-05-27-09-06-05-462554-eisa-153375
[INFO] [launch]: Default logging verbosity is set to INFO
[ERROR] [launch]: Caught exception in launch (see debug for traceback): [Errno 2] No such file or directory: '/home/aw/WVCSC_S2Z_UTB_ARM/install_docker/wvcsc_vehicle_description/share/wvcsc_vehicle_description/config/vehicle_info.param.yaml'
```

原因分析：

**” Autoware 的包名约定 vs 本地包结构不匹配“**

Autoware 的 `planning_simulator.launch.xml` 通过 `vehicle_model` 和 `sensor_model` 参数**自动拼接包名**去查找文件。本地包名要符合这个约定。

```
Autoware 期望的包名约定:

  vehicle_model := wvcsc_vehicle
    └→ 查找 {vehicle_model}_description  → wvcsc_vehicle_description   ✅ 你有
    └→ 查找 {vehicle_model}_launch       → wvcsc_vehicle_launch        ❌ 不存在!

  sensor_model := wvcsc_sensor_kit
    └→ 查找 {sensor_model}_description  → wvcsc_sensor_kit_description ✅ 你有
    └→ 查找 {sensor_model}_launch        → wvcsc_sensor_kit_launch      ❌ 不存在!

```

# 4. planning_simulator.launch.xml 完整数据流分析

##  一、启动链全景
```
ros2 launch autoware_launch planning_simulator.launch.xml \
  vehicle_model:=wvcsc_vehicle \
  sensor_model:=wvcsc_sensor_kit \
  map_path:=/home/robot/autoware_data/maps/sample-map-planning

```

```
                            Autoware 入口
                   planning_simulator.launch.xml
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
   ┌──────────┐       ┌──────────────┐      ┌──────────────┐
   │ 车身模型  │       │  传感器套件   │      │   地图加载    │
   │ vehicle  │       │  sensor_kit  │      │   map_path   │
   └────┬─────┘       └──────┬───────┘      └──────┬───────┘
        │                    │                      │
        ▼                    ▼                      ▼
  wvcsc_vehicle_launch  wvcsc_sensor_kit_launch  autoware_map_loader
  /launch/vehicle.      /launch/sensor_kit.      /map/pointcloud_map
  launch.xml            launch.xml               /map/vector_map
        │                    │                   /map/lanelet2_map
        │                    │
        ▼                    ▼
  ┌────────────────────────────────────────────────────────┐
  │               Autoware 核心算法模块                      │
  │                                                        │
  │  感知 → 定位 → 规划(3级) → 控制 → 仿真器(代替真车)       │
  └────────────────────────────────────────────────────────┘

```

##  二、逐层启动步骤

### 步骤1：车身模型加载

```
vehicle_model:=wvcsc_vehicle
  │
  ├→ $(find wvcsc_vehicle_launch)/launch/vehicle.launch.xml
  │     │
  │     ├→ xacro 加载: $(find wvcsc_sensor_kit_description)/urdf/sensor_kit.xacro
  │     │     │
  │     │     ├→ include $(find wvcsc_vehicle_description)/urdf/vehicle.xacro
  │     │     │     └→ include wvcsc_vehicle.xacro  (子宏定义)
  │     │     │          └→ 定义宏: base_footprint, base_link, wheel, steer_wheel
  │     │     │
  │     │     ├→ include $(find wvcsc_sensor_kit_description)/urdf/sensors.xacro
  │     │     │     ├→ include sensor/lidar/urdf/lidar.xacro  → 定义 sensor_lidar 宏
  │     │     │     └→ include sensor/imu/urdf/imu.xacro      → 定义 sensor_imu 宏
  │     │     │
  │     │     ├→ 调用所有宏 → 生成完整 robot 模型 (8个link)
  │     │     │     base_footprint, base_link,
  │     │     │     left_front_link, right_front_link,
  │     │     │     left_wheel_link, right_wheel_link,
  │     │     │     laser, gyro_link
  │     │     │
  │     │     └→ robot_description 参数
  │     │
  │     └→ robot_state_publisher 节点
  │           ├→ 读取 robot_description
  │           └→ 发布静态 TF: base_footprint→base_link→laser/gyro_link/4轮
  │
  ├→ 加载 vehicle_info.param.yaml → 物理参数注入各 Autoware 节点
  └→ 加载 simulator_model.param.yaml → 仿真器行为参数

```
**产出**: TF 树 (`base_footprint → base_link → ...`), 车辆物理参数

### 步骤2：传感器套件加载
```
sensor_model:=wvcsc_sensor_kit
  │
  ├→ $(find wvcsc_sensor_kit_launch)/launch/sensor_kit.launch.xml
  │     │
  │     ├→ include sensing.launch.xml
  │     │     │
  │     │     ├→ can_bridge.launch.py
  │     │     │     └→ can_bridge_node: VCI_OpenDevice → InitCAN → StartCAN
  │     │     │           发布: can_tx_1, can_tx_2
  │     │     │           订阅: can_rx_1, can_rx_2
  │     │     │
  │     │     ├→ lidar.launch.xml
  │     │     │     ├→ lslidar_cx_launch.py
  │     │     │     │     └→ lslidar_driver_node:
  │     │     │     │           UDP 2368 → 解码 → PCL → /sensing/lidar/pointcloud_raw
  │     │     │     │
  │     │     │     └→ pointcloud_to_laserscan_node:
  │     │     │           /sensing/lidar/pointcloud_raw → /sensing/lidar/scan
  │     │     │           (height过滤 [-0.75,0.5], range [0.5,50])
  │     │     │
  │     │     ├→ imu.launch.xml
  │     │     │     └→ ahrs_driver.launch.py
  │     │     │           └→ ahrs_driver_node:
  │     │     │                 串口 /dev/FDI_IMU_GNSS → CRC校验 → 坐标变换
  │     │     │                 发布: /sensing/imu/tamagawa/imu_raw
  │     │     │                       /sensing/gnss/pose
  │     │     │
  │     │     └→ gnss.launch.xml (预留，GPS已由 ahrs 发布)
  │     │
  │     ├→ wtb_car_driver (底盘驱动节点)
  │     │     订阅: /cmd_vel, can_tx_2
  │     │     发布: /car_odom, can_rx_2, /wtb_car_message
  │     │     (仿真模式下 /cmd_vel 来自仿真器, 不是真底盘)
  │     │
  │     └→ vehicle_interface.launch.xml
  │           └→ wvcsc_vehicle_interface 节点
  │                 订阅: /control/command/control_cmd
  │                      /control/command/gear_cmd
  │                      /car_odom
  │                 发布: /cmd_vel, /run_static
  │                      /vehicle/status/velocity_status
  │                      /vehicle/status/steering_status
  │                      /vehicle/status/control_mode
  │                      /vehicle/status/gear_status

```
**产出**: 传感器原始数据, 底盘控制通道

### 步骤3：地图加载

```
map_path:=/home/robot/autoware_data/maps/sample-map-planning
  │
  ├→ pointcloud_map_loader
  │     读取: *.pcd 点云地图文件
  │     发布: /map/pointcloud_map
  │
  ├→ lanelet2_map_loader
  │     读取: lanelet2_map.osm
  │     发布: /map/vector_map
  │
  ├→ lanelet2_map_visualization
  │     发布: /map/lanelet2_map (可视化标记)
  │
  └→ vector_map_tf_generator
         发布: map→viewer (静态 TF)

```
**产出**: 点云地图, 车道地图

## 三、核心数据流闭环（仿真模式）

这是 `planning_simulator` 的核心：**仿真器代替真车形成闭环**
```
                        ┌─────────────────────────────┐
                        │        Rviz2 用户交互         │
                        │   2D Pose Estimate (初始位姿) │
                        │   Goal 设定 (目标点)          │
                        └──────────┬──────────────────┘
                                   │
                    /initialpose3d │  /planning/.../goal
                                   │
╔══════════════════════════════════╪════════════════════════════════════════╗
║                                  │                                       ║
║  ┌───────────────────────────────┼────────────────────────────────────┐ ║
║  │                         定位层                                     │ ║
║  │                                                                   │ ║
║  │  /sensing/lidar/scan ──→ NDT Scan Matcher ←── /map/pointcloud_map │ ║
║  │         │                      │                                   │ ║
║  │         │              位姿观测 (PoseWithCovarianceStamped)         │ ║
║  │         │                      │                                   │ ║
║  │         ▼                      ▼         /sensing/imu/.../imu_raw │ ║
║  │    ┌────────────────────────────────────────┐                      │ ║
║  │    │            EKF Localizer                │                      │ ║
║  │    │  融合: NDT位姿 + IMU(wz) + 车速         │                      │ ║
║  │    └────────────────┬───────────────────────┘                      │ ║
║  │                     │                                              │ ║
║  │    ┌────────────────▼───────────────────────┐                      │ ║
║  │    │  输出:                                 │                      │ ║
║  │    │  /localization/kinematic_state (Odometry)                     │ ║
║  │    │  /localization/acceleration (Accel)    │                      │ ║
║  │    │  TF: map → base_link                  │                      │ ║
║  │    └────────────────┬───────────────────────┘                      │ ║
║  └─────────────────────┼──────────────────────────────────────────────┘ ║
║                        │                                                ║
║                        ▼                                                ║
║  ┌────────────────────────────────────────────────────────────────────┐ ║
║  │                         感知层 (dummy 模式)                         │ ║
║  │                                                                   │ ║
║  │  autoware_dummy_perception_publisher                               │ ║
║  │     └→ /perception/object_recognition/objects (空列表)             │ ║
║  └────────────────────────────────────────────────────────────────────┘ ║
║                        │                                                ║
║                        ▼                                                ║
║  ┌────────────────────────────────────────────────────────────────────┐ ║
║  │                         规划层 (3级)                                │ ║
║  │                                                                   │ ║
║  │  ① Mission Planner                                                │ ║
║  │     输入: /planning/.../goal + /map/vector_map                    │ ║
║  │     输出: /planning/mission_planning/route (车道级路径)            │ ║
║  │                                                                   │ ║
║  │  ② Behavior Planner                                               │ ║
║  │     输入: route + /perception/objects + /map/vector_map           │ ║
║  │     输出: 行为决策 (LaneFollow) + 参考路径                          │ ║
║  │                                                                   │ ║
║  │  ③ Motion Planner                                                 │ ║
║  │     输入: 参考路径 + 行为 + kinematic_state                        │ ║
║  │     输出: /planning/scenario_planning/trajectory (Trajectory)     │ ║
║  └──────────────────────────────┬─────────────────────────────────────┘ ║
║                                 │                                       ║
║                                 ▼                                       ║
║  ┌────────────────────────────────────────────────────────────────────┐ ║
║  │                         控制层                                     │ ║
║  │                                                                   │ ║
║  │  Trajectory Follower (MPC)                                        │ ║
║  │     输入: /planning/.../trajectory                                │ ║
║  │           /localization/kinematic_state                           │ ║
║  │     输出: /control/command/control_cmd (AckermannControlCommand)  │ ║
║  │           /control/command/gear_cmd                                │ ║
║  └──────────────────────────────┬─────────────────────────────────────┘ ║
║                                 │                                       ║
║              /control/command/control_cmd                               ║
║              /control/command/gear_cmd                                  ║
║                                 │                                       ║
╚═════════════════════════════════╪═══════════════════════════════════════╝
                                  │
                                  ▼
╔═════════════════════════════════╪═══════════════════════════════════════╗
║              仿真闭环 (simple_planning_simulator)                       ║
║                                                                        ║
║  ┌─────────────────────────────────────────────────────────────────┐  ║
║  │  autoware_simple_planning_simulator_node                         │  ║
║  │                                                                  │  ║
║  │  输入:                                                           │  ║
║  │    /control/command/control_cmd (AckermannControlCommand)        │  ║
║  │    /control/command/gear_cmd                                     │  ║
║  │    /initialpose3d (Rviz 2D Pose Estimate)                       │  ║
║  │    simulator_model.param.yaml (延迟、噪声、速度限制)             │  ║
║  │    vehicle_info.param.yaml   (轴距、轮距)                        │  ║
║  │                                                                  │  ║
║  │  内部: 一阶延迟车辆动力学模型                                      │  ║
║  │    velocity += (cmd_velocity - velocity) / time_constant * dt   │  ║
║  │    steer   += (cmd_steer   - steer)   / time_constant * dt     │  ║
║  │    x += velocity * cos(yaw) * dt                                │  ║
║  │    y += velocity * sin(yaw) * dt                                │  ║
║  │    yaw += velocity * tan(steer) / wheel_base * dt               │  ║
║  │                                                                  │  ║
║  │  输出 (模拟传感器):                                               │  ║
║  │    /localization/kinematic_state  ← 模拟位姿 (闭环回定位层!)      │  ║
║  │    /localization/acceleration                                     │  ║
║  │    /vehicle/status/velocity_status ← 模拟速度                     │  ║
║  │    /vehicle/status/steering_status ← 模拟转向角                   │  ║
║  │    /vehicle/status/gear_status                                     │  ║
║  │    /vehicle/status/control_mode                                    │  ║
║  │    TF: map → base_link  ← 模拟位姿的TF表示                         │  ║
║  └─────────────────────────────────────────────────────────────────┘  ║
║                                                                        ║
║  ⚠️ 仿真模式下, wtb_car_driver 和 wvcsc_vehicle_interface             ║
║     虽然启动了, 但 /cmd_vel 被仿真器闭环拦截，不经过CAN到真底盘            ║
║                                                                        ║
╚════════════════════════════════════════════════════════════════════════╝

```

## 四、关键话题数据流图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       仿真闭环话题流                                      │
└─────────────────────────────────────────────────────────────────────────┘

  Rviz2
  │ /initialpose3d              │ /planning/.../goal
  ▼                             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                     Autoware 算法引擎                         │
  │                                                              │
  │  /map/pointcloud_map ──→ NDT匹配 ──→ EKF ──→ /localization/ │
  │  /sensing/lidar/scan ──→           ←── /sensing/imu/raw     │   kinematic_state
  │                                                              │
  │  /localization/kinematic_state ──→ Motion Planner            │
  │  /planning/.../goal            ──→ Mission Planner           │
  │  /perception/objects (dummy)   ──→ Behavior Planner          │
  │                                                              │
  │  /planning/.../trajectory ──→ Trajectory Follower (MPC)     │
  │                                                              │
  │  /control/command/control_cmd  ←── 输出                      │
  │  /control/command/gear_cmd     ←── 输出                      │
  └─────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  simple_planning_simulator (替代真车)                         │
  │                                                              │
  │  接收 control_cmd → 车辆动力学仿真 → 发布:                    │
  │                                                              │
  │  /localization/kinematic_state ──→ 反馈回定位层 (形成闭环)    │
  │  /vehicle/status/velocity_status                              │
  │  /vehicle/status/steering_status                              │
  │  /vehicle/status/gear_status                                  │
  │  /vehicle/status/control_mode                                 │
  │  TF: map → base_link                                          │
  └──────────────────────────────────────────────────────────────┘
         │                                    ▲
         │  /vehicle/status/*                  │  /control/command/*
         ▼                                    │
  ┌────────────────────────────────────────────┴─────────────────┐
  │  wvcsc_vehicle_interface (仿真模式下的作用)                    │
  │                                                              │
  │  仿真器→ /vehicle/status/* → 状态反馈                         │
  │  Autoware→ /control/command/* → 转换为 /cmd_vel              │
  │                                                              │
  │  /car_odom (无数据, 仿真器不产生此话题)                        │
  └──────────────────────────────────────────────────────────────┘

```

## 六、端到端流程：从 Rviz 点目标到仿真车运动

```
时刻T0: 用户在 Rviz 点击 "2D Pose Estimate"
  → /initialpose3d → simple_planning_simulator
  → 仿真器初始化车辆位姿 (x, y, yaw)
  → 发布 TF: map → base_link

时刻T1: 用户在 Rviz 点击 "Goal" 设定目标
  → /planning/mission_planning/goal → Mission Planner
  → Mission Planner 读取 lanelet2_map.osm → 生成车道级 route
  → Behavior Planner 检查 route + /perception/objects(空) → LaneFollow 决策
  → Motion Planner 根据 route + kinematic_state → 生成 trajectory (未来3秒位姿序列)

时刻T2: trajectory 到达 Trajectory Follower (MPC)
  → MPC 计算当前最优控制量:
      longitude.velocity = 0.5 m/s  (逐步加速)
      lateral.steering_tire_angle = 0.0 rad (直行)
  → 发布 /control/command/control_cmd

时刻T3: simple_planning_simulator 收到 control_cmd
  → 一阶延迟动力学:
      actual_velocity ← cmd_velocity (带 0.1s 延迟 + 0.1s 时间常数)
      actual_steer   ← cmd_steer   (带 0.24s 延迟 + 0.27s 时间常数)
  → 阿克曼运动学:
      x += actual_velocity * cos(yaw) * 0.025s
      y += actual_velocity * sin(yaw) * 0.025s
      yaw += actual_velocity * tan(actual_steer) / 0.82 * 0.025s
  → 发布 /localization/kinematic_state (新位姿)
  → 发布 TF: map → base_link (新位姿)
  → Rviz 中车辆模型移动到新位置

时刻T4: 回到 T2, 用新位姿重新规划 → 循环 (25ms = 40Hz)

```
这个闭环一直运行，直到车辆到达目标点（`goal_reached_tolerance` 范围内）或者遇到异常