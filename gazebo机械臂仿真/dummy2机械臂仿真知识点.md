### 1.launch文件的编排
要进行机械臂仿真需要启动多个节点，使用Launch文件进行节点启动的管理和编排。
对于当前项目主要完成机器人在RViz2里的加载、显示和简单控制等功能，主要涉及到==robot_state_publisher==、==rviz2==和==joint_state_publisher_gui==三个功能包中的节点。各节点的作用是：

1.robot_state_publisher: 是ROS 2进行机器人仿真时的核心工具，其以URDF机器人模型为参数作为输入，其在启动后一方面将机器人模型URDF以话题```/robot_description```发布，另一方面接收```/joint_states```话题将机器人的状态从关节位置转化为直角坐标系，按照TF2发布到```/tf```话题，得以向ROS 2加入正确的机器人的坐标系，从而可以在RViz2中正确的可视化。

2.rviz2：用于启动RViz2，进行机器人的可视化。

3.joint_state_publisher_gui: 一个简单的机器人调试工具，能够发布机器人关节位置信息到```/joint_states```话题，从而作为robot_state_publisher的输入信息，用于给定机器人的关节位置。

### 2.urdf文件
#### 2.1.robot元素
机器人模型文件URDF中的根元素必须是 robot ，所有其他元素都必须封装在其中，其内包括link，joint，gazebo等元素。其中gazebo元素用于配置SDF中的属性，用于gazebo仿真。

以下为robot元素的写法
```xml

<?xml version="1.0"?>
<robot name="simple_robot">
   <!-- robot links and joints and more -->
</robot>
```

#### 2.2.link元素
link元素描述具有惯性(inertia)、视觉特征(visual)和碰撞特性(collision)的刚体,只有一个name属性，用于标识link的名称；link元素包含inertia,visual,collision元素。
link的名称base_link是必须的，因为它是机械臂的根连杆，要作为其他连杆的基准。机器人的第一个连杆base_link和joint默认是机器人的原点，即坐标是0,0,0。

以下为link元素的写法
```
<link name="base_link">
  <inertial>
    <origin xyz="2.242591646188575e-07 0.00022711838502637176 0.0543574942352709" rpy="0 0 0"/>
    <mass value="1.2152141810431654"/>
    <inertia ixx="0.002105" iyy="0.002245" izz="0.002436" ixy="-0.0" iyz="-1.1e-05" ixz="0.0"/>
  </inertial>
  <visual>
    <origin xyz="0 0 0" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/base_link.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="0 0 0" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/base_link.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>
```

#### 2.3.joint元素
joint元素描述了连杆间的连接关系，以及关节的运动学和动力学，指定关节的安全极限，包含origin,parent,child,axis,limit等元素。
1.name: joint的名称
2.type: 指定关节的类型，其中类型可以是以下之一:  
  revolute —— 沿轴旋转的铰链关节，其旋转范围由上限和下限指定。
  continuous —— 绕轴旋转的连续旋转关节，没有上限和下限。
  prismatic —— 沿轴平移的平移关节，具有由上限和下限指定的有限范围。
  fixed —— 这实际上不是一个关节，因为它无法移动。所有自由度均已锁定。此类关节不需要 ，axis 、 calibration 、 dynamics 、 limits 或 safety_controller 。
  floating —— 该关节允许所有 6 个自由度的运动。
  planar —— 该关节允许在垂直于轴的平面内运动。

```xml
<joint name="joint1" type="revolute">
  <origin xyz="0.0 0.0 0.096" rpy="0 0 0"/>
  <parent link="base_link"/>
  <child link="link1_1"/>
  <axis xyz="0.0 -0.0 -1.0"/>
  <limit upper="3.001966" lower="-3.001966" effort="100" velocity="100"/>
</joint>
```

```夹爪1（prismatic-沿轴平移）
<joint name="figer1" type="prismatic">
  <origin xyz="0.012824 0.074462 -0.0001" rpy="0 0 0"/>
  <parent link="link6_1"/>
  <child link="figer1_1"/>
  <axis xyz="1.0 -0.0 0.0"/>
  <limit upper="0.028" lower="0.0" effort="100" velocity="100"/>
</joint>
```

```夹爪2（prismatic-沿轴平移）
<joint name="figer2" type="prismatic">
  <origin xyz="-0.012176 0.074462 0.0" rpy="0 0 0"/>
  <parent link="link6_1"/>
  <child link="figer2_1"/>
  <axis xyz="1.0 -0.0 0.0"/>
  <limit upper="0.0" lower="-0.028" effort="100" velocity="100"/>
</joint>
```
以下是joint各子元素的说明：
+ origin: 这是从父连杆到子连杆的变换,joint位于子连杆的原点，是相对于Parent link的frame,而Parent frame=parent_joint frame,也即joint的origin是相对于前一个joint对应frame进行的偏移和旋转。从这一点来看，URDF是以joint为中心。
+ parent: 是父连杆的名称。
+ child: 是子连杆的名称。
+ axis: 对于旋转关节，这是旋转轴；对于平移关节，这是平移轴；对于平面关节，这是表面法线。该轴在关节参考框架中指定。固定关节和浮动关节不使用 axis 字段。
+ limit: 仅旋转关节和平移关节需要,具有以下属性：
    + lower：指定joint下限，旋转关节单位弧度，平移关节以米为单位
    + upper：指定joint上限，旋转关节单位弧度，平移关节以米为单位
    + effort 必要，强制执行最大力，N
    + velocity 必要,强制最大关节速度,rad/s或m/s
+ mimic: 此标签用于指定定义的关节模拟另一个现有关节。该关节的值可以计算为 ： value = multiplier * other_joint_value + offset 。其有三个属性joint(必须)指定了要模仿的关节的名称;multiplier(可选)指定上述公式中的乘法因子;offset(可选)指定上述公式中要添加的偏移量。
### 3.ros2_control
#### 3.1ros2_conrol的介绍
ros2_control是ROS 2中用于控制机器人的一个核心包，它提供了一套标准化的接口和组件来管理机器人硬件的驱动和控制。通过使用ros2_control，开发者可以轻松地集成各种类型的传感器、执行器和控制器到他们的ROS 2应用程序中。
ros2_controllers提供了一组控制器接口和实现，用于控制机器人的运动。这些控制器可以是位置、速度或力/扭矩控制器，并且可以与不同的硬件驱动器配合使用。
ros2_control和 ros2_controllers的结合使用，使得开发者能够方便地实现机器人的控制和运动规划。例如，你可以通过配置位置控制器来控制机械臂的末端执行器到达特定位置，或者使用速度控制器来实现平滑的运动轨迹跟踪。
![[Pasted image 20250805205442.png]]

以下对上图中ros2_control 系统架构的组件进行说明：
==1.控制器管理器 (Controller Manager)==
    Controller Manager 是 ros2_control 框架中的主要组件。 它管理控制器的生命周期、对硬件接口的访问并为 ROS-world 提供服务。
    控制器管理器（CM）连接 ros2_control 框架的控制器层与硬件抽象层，同时作为用户通过ROS服务进行交互的入口点1。CM 实现了无需执行器(excutor)的节点，可集成至自定义架构中。但通常建议使用controller_manager 包内 ros2_control_node 文件提供的默认节点配置。
    CM 一方面管理控制器（加载、激活、停用、卸载）及其所需接口；另一方面通过资源管理器(RM)访问硬件组件（即其接口）。CM 负责匹配控制器所需接口与硬件提供接口，在控制器启用时授予硬件访问权限，并在访问冲突时报错。
    控制循环的执行由 CM 的 update() 方法管理：读取硬件数据 → 更新所有激活控制器的输出 → 将结果写入硬件组件。
    
==2.资源管理器 (Resource Manager)==
   资源管理器（RM）为 ros2_control 框架抽象物理硬件及其驱动程序（称为硬件组件）。RM 通过 pluginlib 库动态加载组件，管理其生命周期及状态/命令接口。RM 的抽象层支持硬件组件的复用（如机械臂与夹爪无需二次开发），并允许状态/命令接口的灵活应用（如电机控制与编码器读取可使用独立通信库）。
  控制循环中，RM 的 read() 和 write() 方法处理与硬件组件的通信。
  
==3.控制器 (Controllers)==
   ros2_control 的控制器基于控制理论：通过比较参考值与测量输出值，依据误差计算系统输入1。控制器派生自 ControllerInterface（ros2_control 中的 controller_interface 包），并通过 pluginlib 导出为插件（参考 ros2_controllers 仓库的 ForwardCommandController 实现）。其生命周期基于 LifecycleNode 类，遵循"节点生命周期设计文档"的状态机规范。
   控制循环执行时，update() 方法被调用，该方法可访问最新硬件状态并写入硬件命令接口。
   
==4.用户接口 (User Interfaces)==
   用户通过控制器管理器的服务与框架交互，服务定义详见 controller_manager_msgs 包的 srv 目录。除命令行直接调用服务外，框架提供集成于 ros2 cli 的友好命令行接口(CLI)，支持自动补全和常用命令，基础命令为 ros2 control（详见 CLI 文档）。
   
==5.硬件组件 (Hardware Components)==
   硬件组件实现与物理硬件的通信，并在框架中代表其抽象实体。组件需通过 pluginlib 导出为插件，由资源管理器动态加载并管理生命周期1。包含三类基础组件：
    * 系统组件 (System)‌: 复杂多自由度硬件（如工业机器人），与执行器组件的核心区别是可使用复杂传动装置（如仿人机械手）。具备读写能力，适用于单一逻辑通信通道（如 KUKA-RSI）。
    * 传感器组件 (Sensor)‌: 感知环境的硬件（如编码器、力矩传感器），仅具备读取能力。
    * 执行器组件 (Actuator)‌：单自由度硬件（如电机、阀门）。若硬件支持模块化设计（如独立 CAN 通信的电机），也可用于多自由度机器人。

#### 3.2ros2_control的安装
```
sudo apt install ros-jazzy-ros2-control -y
sudo apt install ros-jazzy-ros2-controllers -y
```
#### 3.3ros_controller.yaml

```yaml
# This config file is used by ros2_control
controller_manager:
  ros__parameters:
    update_rate: 100  # Hz
    
    dummy2_arm_controller:
      type: joint_trajectory_controller/JointTrajectoryController

    hand_controller:
      type: position_controllers/GripperActionController

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

dummy2_arm_controller:
  ros__parameters:
    joints:
      - joint1
      - joint2
      - joint3
      - joint4
      - joint5
      - joint6
    command_interfaces:
      - position
    state_interfaces:
      - position
      - velocity
hand_controller:
  ros__parameters:
    joints:
      - figer1
      - figer2
```
