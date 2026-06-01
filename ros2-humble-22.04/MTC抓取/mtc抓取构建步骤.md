### 1.创建工作空间
mkdir -p ~/mtc_moveit/src
### 2.进入工作空间并下载源代码
cd ~/mtc_moveit/src
git clone -b humble https://github.com/moveit/moveit_task_constructor.git
### 3.安装依赖并编译
cd ~/mtc_moveit
rosdep install --from-paths src --ignore-src -r -y
source /opt/ros/humble/setup.bash
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release
### 4.进入工作区 src 目录
cd ~/mtc_moveit/src
##克隆整个仓库（或只 sparse‑checkout 子目录）
git clone https://github.com/LitchiCheng/ros2_package.git
mv ros2_package/mtc_tutorial .
##删除多余文件
rm -rf ros2_package
##回到工作区根，安装依赖
或者： ros2 pkg create --build-type ament_cmake --node-name mtc_node mtc_tutorial
### 5.修改 `mtc_demo.launch.py`
打开 `mtc_tutorial/launch/mtc_demo.launch.py`，将原来引用 `moveit2_tutorials` 的 RViz 配置段：
```
# 原版（会报 PackageNotFoundError: moveit2_tutorials）
# rviz_config_file = (
#     get_package_share_directory("moveit2_tutorials") + "/launch/mtc.rviz"
# )
```
替换为指向 `moveit_task_constructor_demo` 包中 `config/mtc.rviz` 的代码：
```
    # RViz 配置文件路径
    rviz_config_file = os.path.join(
        get_package_share_directory("moveit_task_constructor_demo"),
        "config",
        "mtc.rviz"
    )
```
cd ~/mtc_moveit
rosdep install --from-paths src --ignore-src -r -y
source /opt/ros/humble/setup.bash
colcon build --packages-select mtc_tutorial --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
ros2 launch mtc_tutorial mtc_demo.launch.py
ros2 launch mtc_tutorial pick_place_demo.launch.py 
以上会运行熊猫机械臂仿真抓取功能
### 6. 替换dummy2机械臂模型
1.复制dummy2-gripperv2_description与dummy2-gripperv2_moveit_config（路径：/dummy2/ros2/dummy2_ws/src）到工作空间/mtc_moveit/src下

### 7.替换相关源文件
CMakeLists.txt、package.xml、mtc_node.cpp、dummy2-gripperv2.srdf、dummy2-gripperv2.xacro、ros2_controllers.yaml、moveit_controllers.yaml、mtc.rviz、mtc_demo.launch.py、pick_place_demo.launch.py、kinematics.yaml、ompl_planning.yaml

### 8.编译运行
cd ~/mtc_moveit
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
ros2 launch mtc_tutorial mtc_demo.launch.py
ros2 launch mtc_tutorial pick_place_demo.launch.py 
