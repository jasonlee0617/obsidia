### 1.下载相应的gazebo ros包
sudo apt install ros-humble-gazebo-ros
或者：
sudo apt install ros-jazzy-gz-ros2-control -y
### 2.安装相应的缺失包：
##安装 Gazebo 基础组件
sudo apt-get install gazebo11 libgazebo11-dev
##安装 ROS-Gazebo 桥接
sudo apt-get install ros-humble-gazebo-ros-pkgs
##安装可视化插件
sudo apt-get install ros-humble-gazebo-ros

### 启动仿真gazebo.launch
#### 先 source ROS2 和你的 overlay
source /opt/ros/humble/setup.bash
source ~/dummy2/ros2/dummy2_ws/install/setup.bash
#### 再 source Gazebo 的环境文件
source /usr/share/gazebo/setup.sh
#### 这时候再启动 launch，就能正常打开 GUI 了
ros2 launch dummy2-gripperv2_description gazebo.launch.py