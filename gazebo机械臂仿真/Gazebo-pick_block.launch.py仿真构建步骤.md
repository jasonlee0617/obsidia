### GAZEBO安装

#### 更新安装源
```
sudo apt-get update
sudo apt-get install lsb-release wget gnupg
```

#### 安装
```
sudo apt-get install ros-humble-ros-gz
sudo apt install ros-humble-gazebo-ros-pkgs
```

#### 验证
```
ign gazebo shapes.sdf
```
### 提前安装的相关依赖
```
sudo apt update
sudo apt install ros-humble-gazebo-ros-pkgs ros-humble-rviz2
sudo apt install ros-humble-moveit* -y
sudo apt install ros-humble-gz-ros2-control -y
sudo apt install ros-humble-ros-gz-bridge -y
sudo apt-get install ros-humble-ros-gz-sim
sudo apt-get install ros-humble-gazebo-ros2-control
```

### 1.创建机械臂项目
##### 1.1
```
ros2 pkg create --build-type ament_python yolov8_grasping --dependencies rclpy 
```
##### 1.2
然后进入yolov8_grasping文件夹，创建==config rviz launch==文件夹

### 2.moveilt2 setup asistant的配置

#### 1.机械臂描述文件创建
[[moveilt2 setup asistant的配置]]

#### 2.机械臂描述文件修改

##### 2.1 注释掉dummy2-gripperv2.gazebo部分内容

```
<!-- <gazebo>
  <plugin name="control" filename="libgazebo_ros_control.so"/>
</gazebo> -->

<!-- <gazebo reference="camera_1">
  <material>${body_color}</material>
  <mu1>0.2</mu1>
  <mu2>0.2</mu2>
  <self_collide>true</self_collide>
</gazebo> -->

```
##### 2.2 注释掉dummy2-gripperv2.xacro部分内容

```
<!-- <xacro:include filename="$(find dummy2-gripperv2_description)/urdf/dummy2-gripperv2.trans" /> -->

<!-- <link name="camera_1">
  <inertial>
    <origin xyz="-9.097251971671924e-05 0.00021876227592271258 -0.00557732444371406" rpy="0 0 0"/>
    <mass value="0.14371269577068077"/>
    <inertia ixx="1.5e-05" iyy="0.000107" izz="0.000113" ixy="-0.0" iyz="0.0" ixz="0.0"/>
  </inertial>
  <visual>
    <origin xyz="-0.012827 -0.301289 -0.330241" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/camera_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="-0.012827 -0.301289 -0.330241" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/camera_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link> -->

<!-- <joint name="slider_camera" type="prismatic">
  <origin xyz="0.0 0.033212 -0.0235" rpy="0 0 0"/>
  <parent link="link6_1"/>
  <child link="camera_1"/>
  <axis xyz="1.0 -0.0 0.0"/>
  <limit upper="0.0" lower="0.0" effort="100" velocity="100"/>
</joint> -->
```

#### 2.3添加grasp_frame

在**dummy2-gripperv2.xacro**添加**grasp_frame**关节

```
<link name="grasp_frame">
</link>

<joint name="grasp_frame_joint" type="fixed">
  <origin xyz="0.0 0.095 0.0" rpy="0 0 0"/>
  <parent link="link6_1"/>
  <child link="grasp_frame"/>
</joint>
```

### 3.yolov8_grasping功能包
#### 3.1 config文件
##### 3.1.1
在yolov8_grasping功能包中的config文件夹创建一个==box.urdf==文件，表示要抓取的方块
[[box.urdf]]
##### 3.1.2
在yolov8_grasping功能包的config文件夹中创建一个==case.urdf==文件，作为放置小方块的盒子
[[case.urdf]]
##### 3.1.3 创建yolov8_grasping_gazebo.friction.urdf.xacro

在yolov8_grasping功能包的config目录下创建==yolov8_grasping_gazebo.friction.urdf.xacro==

==yolov8_grasping_gazebo.friction.urdf.xacro与dummy2-gripperv2.urdf.xacro区别==
###### 加入内容：
```
    <gazebo>
        <plugin filename="gz_ros2_control-system" name="gz_ros2_control::GazeboSimROS2ControlPlugin">
            <parameters>$(find dummy2-gripperv2_moveit_config)/config/ros2_controllers.yaml</parameters>
        </plugin>
    </gazebo>
```

###### 修改内容：

```
    <xacro:include filename="dummy2-gripperv2.ros2_control.xacro" />
    <xacro:dummy2-gripperv2_ros2_control name="FakeSystem" initial_positions_file="$(arg initial_positions_file)"/>
```

**替换为：**

```
    <xacro:include filename="yolov8_grasping_gazebo.friction.xacro" />
    <xacro:dummy2-gripperv2_ros2_control name="GazeboSystem" initial_positions_file="$(arg initial_positions_file)"/>
```
##### 3.1.4 创建yolov8_grasping_gazebo.friction.xacro

在yolov8_grasping功能包的config目录下创建==yolov8_grasping_gazebo.friction.xacro==

==yolov8_grasping_gazebo.friction.xacro与dummy2-gripperv2.ros2_control.xacro区别==

```
          <hardware>
                <plugin>mock_components/GenericSystem</plugin>
            </hardware>
```

**替换为**

```
            <hardware>
                <plugin>gz_ros2_control/GazeboSimSystem</plugin>
            </hardware>
```

#### 3.2 launch文件
在yolov8_grasping功能包==创建launch文件夹==并在其中创建一个名为==`pick_block.launch.py`==文件
[[pick_block.launch.py]]

### 4.编写机械臂控制节点

可以利用MoveIt2的Python API编写一个机械臂控制节点，实现机械臂抓取和放置小方块的节点。

在yolov8_grasping功能包中的**yolov8_grasping**文件夹创建文件**pick_drop.py、pick_drop.py**

[[pick_drop_node.py]]
[[pick_drop_ik_node.py]]


### 5.修改yolov8_grasping功能包内的setup.py内容
#### 6.1识别特定文件格式

在**data_files**中加入以下代码：

```
        (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
        (os.path.join('share', package_name, 'config'), glob('config/*.yaml')),
        (os.path.join('share', package_name, 'rviz'), glob('rviz/*.rviz')), 
        (os.path.join('share', package_name, 'config'), glob('config/*.sdf')),
        (os.path.join('share', package_name, 'config'), glob('config/*.urdf')),
        (os.path.join('share', package_name, 'config'), glob('config/*.xacro'))
```

#### 6.2识别运行节点
在=='console_scripts':== 中加入如下内容

```
            'pick_drop = yolov8_grasping.pick_drop_node:main',
            'pick_drop_ik = yolov8_grasping.pick_drop_ik_node:main',
```

### 6.编译运行

```
colcon build --symlink-install
source install/setup.bash
ros2 launch yolov8_grasping pick_block.launch.py
```






