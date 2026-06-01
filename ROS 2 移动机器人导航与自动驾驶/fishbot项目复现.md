- [[#在RVIZ中显示机器人模型|在RVIZ中显示机器人模型]]
- [[#仿真|仿真]]
	- [[#仿真#错误解决|错误解决]]
	- [[#仿真#打开bashrc|打开bashrc]]
- [[#注释代码|注释代码]]
	- [[#注释代码#文件最后添加如下内容|文件最后添加如下内容]]
- [[#仿真+建图|仿真+建图]]
	- [[#仿真+建图#其中键盘控制方式如下|其中键盘控制方式如下]]
- [[#仿真 + Nav2 导航|仿真 + Nav2 导航]]
	- [[#仿真 + Nav2 导航#然后在 RViz 里：|然后在 RViz 里：]]
	- [[#仿真 + Nav2 导航#打开RVI界面后左边页面栏中出现报错|打开RVI界面后左边页面栏中出现报错]]
	- [[#仿真 + Nav2 导航#点击RVIZ中的“2D Pose Estimate”估计小车本体的初始位置|点击RVIZ中的“2D Pose Estimate”估计小车本体的初始位置]]
	- [[#仿真 + Nav2 导航#然后点击“Nav2 Goal”小车需要到达的目的地|然后点击“Nav2 Goal”小车需要到达的目的地]]
- [[#3.1 `gazebo.launch.py` 做了什么？|3.1 `gazebo.launch.py` 做了什么？]]
	- [[#3.1 `gazebo.launch.py` 做了什么？#3.1.1FishBot 的 `gazebo.launch.py` 做了三件核心事情：|3.1.1FishBot 的 `gazebo.launch.py` 做了三件核心事情：]]
	- [[#3.1 `gazebo.launch.py` 做了什么？#3.1.2Gazebo 为什么能控制小车？|3.1.2Gazebo 为什么能控制小车？]]
- [[#3.2 fishbot_cartographer:负责建图|3.2 fishbot_cartographer:负责建图]]
	- [[#3.2 fishbot_cartographer:负责建图#3.2.1Cartographer 吃什么数据？|3.2.1Cartographer 吃什么数据？]]
	- [[#3.2 fishbot_cartographer:负责建图#3.2.2 建图数据流|3.2.2 建图数据流]]
	- [[#3.2 fishbot_cartographer:负责建图#3.2.3 SLAM|3.2.3 SLAM]]
	- [[#3.2 fishbot_cartographer:负责建图#3.2.4Cartographer 内部建图原理简化版|3.2.4Cartographer 内部建图原理简化版]]
	- [[#3.2 fishbot_cartographer:负责建图#3.2.5Cartographer 输入和输出|3.2.5Cartographer 输入和输出]]
	- [[#3.2 fishbot_cartographer:负责建图#3.2.6三个节点分别干什么？|3.2.6三个节点分别干什么？]]
		- [[#3.2.6三个节点分别干什么？#3.2.6.1cartographer_node|3.2.6.1cartographer_node]]
		- [[#3.2.6三个节点分别干什么？#3.2.6.2 cartographer_occupancy_grid_node|3.2.6.2 cartographer_occupancy_grid_node]]
		- [[#3.2.6三个节点分别干什么？#3.2.6.3 rviz2|3.2.6.3 rviz2]]
- [[#3.3fishbot_navigation2:负责自主导航|3.3fishbot_navigation2:负责自主导航]]
	- [[#3.3fishbot_navigation2:负责自主导航#3.3.1 `fishbot_navigation2` 在整个系统中的位置|3.3.1 `fishbot_navigation2` 在整个系统中的位置]]
	- [[#3.3fishbot_navigation2:负责自主导航#3.3.2 Nav2 里面有哪些核心模块|3.3.2 Nav2 里面有哪些核心模块]]
	- [[#3.3fishbot_navigation2:负责自主导航#3.3.3Nav2 的完整数据流|3.3.3Nav2 的完整数据流]]
- [[#3.4 建图和导航是两个阶段，不要混在一起|3.4 建图和导航是两个阶段，不要混在一起]]
	- [[#3.4 建图和导航是两个阶段，不要混在一起#3.4.1 阶段 A：建图|3.4.1 阶段 A：建图]]
	- [[#3.4 建图和导航是两个阶段，不要混在一起#3.4.2 阶段 B：保存地图|3.4.2 阶段 B：保存地图]]
	- [[#3.4 建图和导航是两个阶段，不要混在一起#3.4.3 导航|3.4.3 导航]]
- [[#3.5 应该掌握的一套调试命令|3.5 应该掌握的一套调试命令]]
	- [[#3.5 应该掌握的一套调试命令#3.5.1 看节点|3.5.1 看节点]]
	- [[#3.5 应该掌握的一套调试命令#3.5.2 看话题|3.5.2 看话题]]
	- [[#3.5 应该掌握的一套调试命令#3.5.3 看 TF|3.5.3 看 TF]]
	- [[#3.5 应该掌握的一套调试命令#3.5.4 看 `/cmd_vel`|3.5.4 看 `/cmd_vel`]]
	- [[#3.5 应该掌握的一套调试命令#3.5.5 看生命周期状态|3.5.5 看生命周期状态]]

# 项目地址：https://github.com/fishros/fishbot

# 项目框架
```
1. 机器人描述：URDF、mesh、joint、sensor、Gazebo 插件
2. Gazebo 仿真：世界文件、小车生成、差速驱动、雷达、IMU -fishbot_description
3. SLAM 建图：Cartographer -fishbot_cartographer
4. 地图保存：map_server / map_saver
5. 自主导航：Nav2 -fishbot_navigation2
6. 可视化：RViz2
```


# 1.下载编译
```
git clone --recursive https://github.com/fishros/fishbot.git -b humble
cd fishbot
rosdep install --from-paths src -y
colcon build --symlink-install
```
# 2.运行测试
## 在RVIZ中显示机器人模型
```
source install/setup.bash
ros2 launch fishbot_description display_rviz2.launch.py
```

## 仿真
```
source install/setup.bash
ros2 launch fishbot_description gazebo.launch.py
```

### 错误解决
![[Pasted image 20260509100954.png]]

### 打开bashrc
```
code ~/.bashrc
```
## 注释代码
![[Pasted image 20260509101546.png]]

### 文件最后添加如下内容
```
# ===== ROS 2 Humble + Gazebo Classic 11 + FishBot =====

# Reset Gazebo environment variables to avoid repeated paths

unset GAZEBO_MODEL_PATH

unset GAZEBO_RESOURCE_PATH

unset GAZEBO_PLUGIN_PATH

  

# Gazebo Classic 11

if [ -f /usr/share/gazebo/setup.sh ]; then

source /usr/share/gazebo/setup.sh

fi

  

# ROS 2 Humble

if [ -f /opt/ros/humble/setup.bash ]; then

source /opt/ros/humble/setup.bash

fi

  
  

# Clean Gazebo paths after all source commands

clean_gazebo_paths() {

export GAZEBO_MODEL_PATH="$(echo "$GAZEBO_MODEL_PATH" \

| tr ':' '\n' \

| sed '/^$/d' \

| grep -v '^/opt/ros/humble/share$' \

| awk '!seen[$0]++' \

| paste -sd: -)"

  

export GAZEBO_RESOURCE_PATH="$(echo "$GAZEBO_RESOURCE_PATH" \

| tr ':' '\n' \

| sed '/^$/d' \

| grep -v '^/opt/ros/humble/share$' \

| awk '!seen[$0]++' \

| paste -sd: -)"

  

export GAZEBO_PLUGIN_PATH="$(echo "$GAZEBO_PLUGIN_PATH" \

| tr ':' '\n' \

| sed '/^$/d' \

| awk '!seen[$0]++' \

| paste -sd: -)"

}

  

clean_gazebo_paths

  

# Input method

export GTK_IM_MODULE=fcitx

export QT_IM_MODULE=fcitx

export XMODIFIERS=@im=fcitx

  

# Force Gazebo / RViz / OpenGL applications to use NVIDIA GPU

export __NV_PRIME_RENDER_OFFLOAD=1

export __GLX_VENDOR_LIBRARY_NAME=nvidia

export __VK_LAYER_NV_optimus=NVIDIA_only

export LIBGL_ALWAYS_SOFTWARE=0

unset MESA_LOADER_DRIVER_OVERRIDE

  

# ===== End ROS 2 Humble + Gazebo Classic 11 + FishBot =====

# Auto-clean Gazebo paths after sourcing ROS / colcon setup files

source() {

builtin source "$@"

local ret=$?

  

case "$1" in

*/setup.bash|*/local_setup.bash|setup.bash|local_setup.bash|install/setup.bash)

if declare -F clean_gazebo_paths >/dev/null; then

clean_gazebo_paths

fi

;;

esac

  

return $ret

}
```


## 仿真+建图
```
source install/setup.bash
ros2 launch fishbot_description gazebo.launch.py
ros2 launch fishbot_cartographer cartographer.launch.py
ros2 run teleop_twist_keyboard teleop_twist_keyboard
ros2 run nav2_map_server map_saver_cli -f ~/fishbot/src/fishbot_navigation2/maps/fishbot_map2
```

### 其中键盘控制方式如下
```
u    i    o
j    k    l
m    ,    .

```

```
i    前进
,    后退
j    原地左转
l    原地右转
k    停止
u    左前方
o    右前方
m    左后方
.    右后方

```

## 仿真 + Nav2 导航

```
ros2 launch fishbot_description gazebo.launch.py
ros2 launch fishbot_navigation2 navigation2.launch.py use_sim_time:=True
```

### 然后在 RViz 里：
```
2D Pose Estimate
Nav2 Goal
```

### 打开RVI界面后左边页面栏中出现报错
![[Pasted image 20260509112743.png]]

### 点击RVIZ中的“2D Pose Estimate”估计小车本体的初始位置
![[Pasted image 20260509112836.png]]

### 然后点击“Nav2 Goal”小车需要到达的目的地


![[Pasted image 20260509112910.png]]

# 3.项目讲解
## 3.1 `gazebo.launch.py` 做了什么？

### 3.1.1FishBot 的 `gazebo.launch.py` 做了三件核心事情：
```
1. 启动 Gazebo，并加载 fishbot.world
2. 调用 spawn_entity.py，把 fishbot_gazebo.urdf 生成到 Gazebo
3. 启动 robot_state_publisher，发布机器人 TF

```

源码里可以看到它使用：
```
gazebo --verbose -s libgazebo_ros_init.so -s libgazebo_ros_factory.so fishbot.world
```
然后用 `gazebo_ros` 的 `spawn_entity.py` 从 URDF 文件生成名为 `fishbot` 的实体

所以当运行:
```
ros2 launch fishbot_description gazebo.launch.py
```
启动了一组进程：
```
gazebo
spawn_entity.py
robot_state_publisher
```

### 3.1.2Gazebo 为什么能控制小车？
因为 URDF 里配置了 Gazebo 插件:
```
[diff_drive]: Subscribed to [/cmd_vel]
[diff_drive]: Advertise odometry on [/odom]
```

说明 Gazebo 里的差速驱动插件正在做这件事：
```
订阅 /cmd_vel
    ↓
控制左右轮转动
    ↓
让 Gazebo 里的小车运动
    ↓
发布 /odom
    ↓
发布 odom -> base_footprint 的 TF

```
**所以 `/cmd_vel` 是底盘控制入口,订阅者就是 Gazebo 里的 diff_drive 插件**

```
第 1 步：找到 fishbot_description 包的位置

第 2 步：找到 world 文件
    fishbot_description/world/fishbot.world

第 3 步：找到 URDF 文件
    fishbot_description/urdf/fishbot_gazebo.urdf

第 4 步：启动 Gazebo
    gazebo --verbose -s libgazebo_ros_init.so -s libgazebo_ros_factory.so fishbot.world

第 5 步：启动 robot_state_publisher
    读取 URDF
    发布机器人 link/joint 的 TF

第 6 步：启动 spawn_entity.py
    把 URDF 机器人生成到 Gazebo 里面

第 7 步：Gazebo 读取 URDF 里的插件
    diff_drive 插件
    laser 插件
    imu 插件

第 8 步：小车开始发布和订阅话题
    订阅 /cmd_vel
    发布 /odom
    发布 /scan
    发布 /imu
    发布 /tf

```

**Gazebo环境启动关键插件：**

**-s libgazebo_ros_init.so:这个插件让 Gazebo 初始化 ROS 2 支持,没有它，Gazebo 很难正常接入 ROS 2 通信系统**
```
libgazebo_ros_init.so
    让 Gazebo 认识 ROS 2
```

**-s libgazebo_ros_factory.so:这个插件提供 `/spawn_entity` 服务**
```
libgazebo_ros_factory.so
    允许 ROS 2 动态往 Gazebo 里面生成机器人

```

## 3.2 fishbot_cartographer:负责建图

运行：
```
ros2 launch fishbot_cartographer cartographer.launch.py
```

它启动的是：
```
cartographer_node
cartographer_occupancy_grid_node
rviz2
```

### 3.2.1Cartographer 吃什么数据？
Cartographer 主要需要：
```
/scan     激光雷达
/odom     里程计
/tf       坐标变换
/clock    仿真时间
```

FishBot 的 `fishbot_2d.lua` 里配置了：
```
map_frame = "map"
tracking_frame = "base_link"
published_frame = "odom"
odom_frame = "odom"
provide_odom_frame = false
use_odometry = true
num_laser_scans = 1
```
也就是说，它使用里程计和一个 2D 雷达进行建图，并参与维护 `map`、`odom`、`base_link` 之间的关系

### 3.2.2 建图数据流

```
Gazebo 中的小车
    ↓
发布 /scan、/odom、/tf、/clock
    ↓
Cartographer 接收这些数据
    ↓
进行 2D SLAM
    ↓
生成子图 submaps
    ↓
cartographer_occupancy_grid_node 把子图转换成 /map
    ↓
RViz 显示地图
    ↓
map_saver_cli 保存地图

```

### 3.2.3 SLAM

**SLAM 全称是：Simultaneous Localization and Mapping-同步定位与地图构建**

```
机器人一边不知道地图长什么样，
一边也不知道自己在哪里，
但它要通过传感器数据，
同时估计“地图”和“自己在地图中的位置”。
```

Cartographer 主要用这些数据：
```
/scan
    激光雷达数据，告诉机器人周围障碍物的距离

/odom
    里程计数据，告诉机器人自己大概移动了多少

/tf
    坐标变换，告诉系统 laser_link、base_link、odom 等坐标系之间的关系

/clock
    Gazebo 仿真时间
```

然后 Cartographer 逐步构建：
```
/map
    栅格地图
```

### 3.2.4Cartographer 内部建图原理简化版

```
1. 机器人收到一帧 /scan

2. 根据 /odom 预测机器人大概移动到了哪里

3. 根据 TF 把雷达数据转换到机器人坐标系

4. 用 scan matching 把当前雷达和局部地图对齐

5. 得到更准确的机器人位姿

6. 把这帧雷达数据插入当前 submap

7. 机器人继续移动，持续生成新的 submap

8. 系统在后台检查有没有回到旧地方

9. 如果检测到回环，就优化整条轨迹

10. occupancy_grid_node 把 submaps 转成 /map

```

可以把 Cartographer 想成两部分：
```
前端：
    实时处理雷达，构建局部地图

后端：
    做回环检测和全局优化，减少累计误差

```

### 3.2.5Cartographer 输入和输出

输入
建图时需要这些：
```
/scan
    sensor_msgs/msg/LaserScan
    激光雷达

/odom
    nav_msgs/msg/Odometry
    轮式里程计

/tf
    tf2_msgs/msg/TFMessage
    动态坐标变换

/tf_static
    tf2_msgs/msg/TFMessage
    静态坐标变换

/clock
    rosgraph_msgs/msg/Clock
    Gazebo 仿真时间

```

输出
```
/map
    nav_msgs/msg/OccupancyGrid
    栅格地图

/tf
    map -> odom
    全局地图坐标到里程计坐标的修正

/submap_list
    Cartographer 内部子图列表

/trajectory_node_list
    轨迹节点列表

/landmark_poses_list
    路标相关信息，FishBot 不重点使用

```

### 3.2.6三个节点分别干什么？
#### 3.2.6.1cartographer_node

它是建图核心节点,它负责：
```
订阅 /scan
订阅 /odom
读取 /tf
使用 fishbot_2d.lua 参数
进行 2D SLAM
生成 submaps
发布 map -> odom 相关 TF
发布 Cartographer 相关内部话题

```

可以理解为：
```
cartographer_node = SLAM 算法大脑
```

#### 3.2.6.2 cartographer_occupancy_grid_node

的作用是：
```
把 Cartographer 生成的 submaps 转换成普通 ROS 地图 /map
```

所以：
```
cartographer_node
    生成 submaps

cartographer_occupancy_grid_node
    把 submaps 转换成 /map

```

#### 3.2.6.3 rviz2

RViz 负责显示：
```
机器人模型
雷达数据
TF
Cartographer 子图
/map 地图
机器人轨迹
```

## 3.3fishbot_navigation2:负责自主导航

运行：
```
ros2 launch fishbot_navigation2 navigation2.launch.py use_sim_time:=True
```

这个 launch 文件不是自己手写所有 Nav2 节点，而是调用了 Nav2 官方 `nav2_bringup` 里的 `bringup_launch.py`，并传入三个关键参数：
```
map
use_sim_time
params_file
```

所以它的本质是：
```
FishBot 自己准备地图和参数
    ↓
交给 Nav2 bringup_launch.py
    ↓
Nav2 启动地图服务器、AMCL、规划器、控制器、行为树等
```

### 3.3.1 `fishbot_navigation2` 在整个系统中的位置

总框架：
```
Gazebo 仿真
    ↓
发布 /scan、/odom、/tf、/clock
    ↓
fishbot_navigation2
    ↓
map_server 加载地图
amcl 定位
planner_server 规划全局路径
controller_server 生成速度
bt_navigator 组织导航行为
costmap 生成代价地图
    ↓
发布 /cmd_vel
    ↓
Gazebo diff_drive 插件
    ↓
小车运动

```
**它提供移动机器人的定位、规划、控制、感知、可视化等能力，并通过行为树协调多个模块化服务器，最终输出机器人底盘可执行的速度命令。Nav2 的预期输入包括 TF、地图、行为树 XML、传感器数据，输出是有效的速度命令**

**一句话总结：**
**fishbot_navigation2 不是自己写导航算法，**
**它是把 FishBot 的地图、参数、仿真时间配置交给 Nav2，**
**然后由 Nav2 完成定位、规划、避障和控制**

**文件关键作用：**
```
launch/navigation2.launch.py
    启动 Nav2

param/fishbot.yaml
    Nav2 参数文件

maps/fishbot_map.yaml + fishbot_map.pgm
    已保存的地图

FishBot 只是准备好：
    地图
    参数
    仿真时间
然后调用 Nav2 官方 bringup_launch.py。
```
### 3.3.2 Nav2 里面有哪些核心模块
| 模块                | 作用                   | 你可以怎么理解              |
| ----------------- | -------------------- | -------------------- |
| map_server        | 加载 `.yaml + .pgm` 地图 | 把静态地图发布成 `/map`      |
| amcl              | 定位                   | 根据地图和雷达估计机器人在地图中的位置  |
| planner_server    | 全局规划                 | 从当前位置到目标点规划一条路径      |
| controller_server | 局部控制                 | 跟踪路径，实时生成 `/cmd_vel` |
| bt_navigator      | 行为树导航器               | 组织整个导航流程             |
| behavior_server   | 恢复行为                 | 负责恢复行为，例如旋转、后退、等待    |
| local_costmap     | 局部代价地图               | 机器人附近障碍物             |
| global_costmap    | 全局代价地图               | 整张地图上的通行代价           |
| lifecycle_manager | 生命周期管理               | 启动、激活、关闭 Nav2 节点     |
**AMCL 是 Nav2 里常见的定位模块，官方文档说它使用粒子滤波，在已知地图和 2D 激光雷达基础上估计机器人的位置和朝向**

### 3.3.3Nav2 的完整数据流

```
你在 RViz 点击 Nav2 Goal
    ↓
RViz 发送 NavigateToPose action
    ↓
bt_navigator 接收导航任务
    ↓
bt_navigator 请求 planner_server 规划路径
    ↓
planner_server 使用 global_costmap 生成全局路径
    ↓
bt_navigator 请求 controller_server 跟踪路径
    ↓
controller_server 使用 local_costmap 计算速度
    ↓
controller_server 发布 /cmd_vel
    ↓
Gazebo diff_drive 插件接收 /cmd_vel
    ↓
小车运动
    ↓
Gazebo 更新 /odom、/scan、/tf
    ↓
amcl 更新机器人定位
    ↓
Nav2 继续闭环控制

```
## 3.4 建图和导航是两个阶段，不要混在一起
### 3.4.1 阶段 A：建图

你还没有地图，目标是生成地图
```
ros2 launch fishbot_description gazebo.launch.py
ros2 launch fishbot_cartographer cartographer.launch.py
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -r cmd_vel:=/cmd_vel
```

数据流是：
```
Gazebo 小车
    ↓ 发布
/scan、/odom、/tf
    ↓
Cartographer
    ↓
/map

```

控制小车走一圈，地图逐渐形成

### 3.4.2 阶段 B：保存地图

建完图后保存：
```
ros2 run nav2_map_server map_saver_cli -f ~/fishbot/src/fishbot_navigation2/maps/fishbot_map

```

会生成：
```
fishbot_map.yaml
fishbot_map.pgm
```

然后重新编译：
```
cd ~/fishbot
colcon build --symlink-install
source install/setup.bash
```

### 3.4.3 导航

导航时已经有地图了，所以不再启动 Cartographer
启动：
```
ros2 launch fishbot_description gazebo.launch.py
ros2 launch fishbot_navigation2 navigation2.launch.py use_sim_time:=True
```

然后在 RViz 里：
```
1. 2D Pose Estimate 设置初始位姿
2. Nav2 Goal 设置目标点
3. Nav2 才开始发布 /cmd_vel
```

这时数据流是：
```
map_server 发布 /map
Gazebo 发布 /scan /odom /tf
AMCL 根据 map + scan + odom 定位
Planner Server 规划路径
Controller Server 发布 /cmd_vel
Gazebo 小车运动
```

## 3.5 应该掌握的一套调试命令

### 3.5.1 看节点
```
ros2 node list
```

### 3.5.2 看话题
```
ros2 topic list
```

### 3.5.3 看 TF

安装工具：
```
sudo apt install ros-humble-tf2-tools
```

生成 TF 树：
```
ros2 run tf2_tools view_frames
```

然后打开生成的 PDF：
```
evince frames.pdf
```

### 3.5.4 看 `/cmd_vel`

```
ros2 topic echo /cmd_vel

```

### 3.5.5 看生命周期状态

Nav2 大量节点是 lifecycle node，不是普通节点

查看：
```
ros2 lifecycle nodes
```

```
ros2 node list
ros2 node info
ros2 topic list
ros2 topic echo
ros2 topic info
ros2 service list
ros2 action list
ros2 param list
ros2 launch

```