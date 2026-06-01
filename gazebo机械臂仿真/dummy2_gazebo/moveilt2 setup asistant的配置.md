### moveilt2 setup asistant的配置
MoveIt2是ROS 2的一个框架，包含了一组相关的功能包，在进行前需要安装，命令如下：·
```
```bash

sudo apt install ros-humble-moveit ros-humble-moveit-setup-assistant -y
```
#### **步骤 1：启动配置助手**
启动 MoveIt 设置助手，命令如下：
```
export QT_QPA_PLATFORM=xcb
source install/setup.bash
ros2 launch moveit_setup_assistant setup_assistant.launch.py
```
选择 **Create New MoveIt Configuration Package** → 点击 **Browse** 加载 URDF 文件 → **Load Files**。
#### **步骤 2：关键配置项**
#### 1.生成自碰撞矩阵（Self-Collisions ）
默认的自碰撞矩阵生成器可以通过禁用机器人上已知安全的连杆对的碰撞检查来帮助缩短运动规划时间。  
可以设置采样密度，该密度决定了有多少个随机机器人位置需要进行自碰撞检查。最后点击生成自碰撞矩阵按钮==Generate Collision Matrix==，设置助手将花费几秒钟来计算自碰撞矩阵，其中包括检查哪些链接对可以安全地禁用碰撞检查。
- 点击左侧 **Self-Collisions** → **Generate Collision Matrix**（默认采样密度 10000）。
- -_作用_：禁用总碰撞或相邻链的碰撞检测，提升规划效率。
![[Pasted image 20250805192141.png]]
#### 2.添加虚拟关节 (虚拟关节)
虚拟关节主要用于连接机器人与外界。对于 robot_arm机械臂，定义固定虚拟关节是可选的。我们将定义一个fixed虚拟关节，用于连接base_link连杆与world坐标系的相对位置。这个虚拟关节表示手臂的底部在世界坐标系中保持静止。
此处各配置参数为：
	* Virtual Joint Name: virtual_joint
* Child Link: base_link
* Parent Frame: world
* Joint Type: fixed
![[Pasted image 20250805192329.png]]
#### 3.添加规划组（Planning Groups）
MoveIt中的规划组在语义上描述机器人的不同部分，例如手臂或末端执行器，按照功能以促进运动规划。  
这里添加两个规划组：定义位的规划组dummy2_arm和夹爪的规划组hand。
##### 3.1首先添加arm规划组
点击==Add Group==按钮，选择 ==kdl_kinematics_plugin/KDLKinematicsPlugin== 作为运动学解算器，因为这是 MoveIt 的默认设置,让 Kin.Search Resolution 和 Kin.Search Timeout 保持其默认值，默认规划器设置为==BiRRT==
![[Pasted image 20250805192636.png]]

完成后点击 Add Joints（添加关节”）按钮，将virtual_joint到joint6都添加，完成后点击Save（保存）按钮。
![[Pasted image 20250805192821.png]]
##### 3.2添加末端执行器组hand
按照上述办法添加末端执行器组hand，配置如下图所示。
![[Pasted image 20250805192940.png]]
点击 “添加关节” 按钮，把finger1和finger2添加进来，保存。
![[Pasted image 20250805193007.png]]
上述两个规划组添加完成后，添加手臂和手组后，自定义组列表应如下所示。
![[Pasted image 20250805193043.png]]
#### 4.添加机器臂姿态（Robot Poses ）
MoveIt Setup Assistant允许将预定义姿态添加到机器人配置中，这对于定义特定的初始姿态或准备姿态非常有用。点击 **Add Pose** → 拖动关节滑块或输入弧度值 → **Save**（如 `home` 位姿）。
设置dummy2_arm规划组的home姿，如下图所示。
![[Pasted image 20250805193224.png]]
设置hand规划组的close和open姿态，如下图所示。
![[Pasted image 20250805193331.png]]

![[Pasted image 20250805193432.png]]
姿态配置完成的效果如下：
![[Pasted image 20250805193520.png]]
#### 5.标记末端执行器(End Effectors )
现在已经将dummy2_arm的手添加到规划组，可以将其指定为末端执行器。通过将组指定为末端执行器，MoveIt可以对其执行某些特殊操作。例如，末端执行器可用于在执行拾取和放置任务时将物体附着到手臂上。
- 若有夹爪：
        - **Name**: `gripper`
        - **End Effector Group**: 选 `gripper_group`
        - **Parent Link**: 夹爪父连杆（如 `link6`）。
配置参数如下图所示。
![[Pasted image 20250805193818.png]]
#### 6.添加被动关节
“被动关节” 旨在指定机器人中可能存在的任何被动关节。这些关节是非驱动关节，这意味着它们无法直接控制。指定被动关节非常重要，这样规划器才能感知到它们的存在，并避免为其进行规划。如果规划器不知道被动关节的存在，它们可能会尝试规划涉及移动被动关节的轨迹，从而导致规划无效。
如果没有任何被动关节，可以忽略。

#### 7.ros2_control URDF修改
ros2_control URDF修改窗格有助于修改机器人URDF以配合使用ros2_control。
command_interface标签定义了可发送用于控制关节的 command_interface类型。state_interface 定义了可从关节读取的状态信息类型。
对于Command interface勾选position，而对State interface勾选position和volcity，完成后，点击==add interface==按钮确认修改，如下图所示。
![[Pasted image 20250805202109.png]]
#### 8.ros2_contorllers
ros2_controllers可用于自动生成模拟控制器来驱动机器人关节。
ros2 control是一个用于实时控制机器人的框架，旨在管理和简化新机器人硬件的集成。
点击Auto Add JointTrajectoryControllers（自动添加关节轨迹控制器按钮），会自动生成两个控制器，分别是dummy2_controller和hand_controller。

8.1.双击dummy2_controller，进入选项卡内，更改controller_type为==joint_trajectory_controller/JointTrajectoryController==，之后保存。注意：==一定要进入该选项卡后一定要保存，否则生成的配置文件中不会有dummy2_controller控制器的配置。（这是一个软件Bug）==

8.2.双击hand_controller,更改controller_type修改为position_controllers/GripperActionController，之后保存

完成后的配置如下图所示：
  ![[Pasted image 20250805202617.png]]
#### 9.配置MoveIt Controllers
MoveIt 需要带有 FollowJointTrajectoryAction接口的轨迹控制器来执行规划的轨迹。该接口将生成的轨迹发送给机器人ROS2控制器。
与==8.ros2_contorllers==一样，点击自动生成后，点击每个控制器，点击保存，同时修改hand_controller的Controller Type为==GripperCommand== 点击保存。
![[Pasted image 20250805202837.png]]
#### 10.Perception感知
其中的perception选项卡用于配置机器人使用的3D传感器。这些设置保存在名为 sensor_3d.yaml的YAML配置文件中。
![[Pasted image 20250805203303.png]]

| 配置项                              | 值                                               |
| -------------------------------- | ----------------------------------------------- |
| **3D 传感器类型**                     | Depth Map                                       |
| **Image Topic**                  | `/head_mount_kinect/depth_registered/image_raw` |
| **Queue Size**                   | 5                                               |
| **Near Clipping Plane Distance** | 0.3                                             |
| **Far Clipping Plane Distance**  | 5.0                                             |
| **Shadow Threshold**             | 0.2                                             |
| **Padding Offset**               | 0.03                                            |
| **Padding Scale**                | 4.0                                             |
| **Filtered Cloud Topic**         | `filtered_cloud`                                |
| **Max Update Rate**              | 1.0                                             |
#### 11.Launch Files
在 “Launch Files” 窗格中，您可以查看将要生成的Launch文件列表。默认选项通常足够，但如果您对应用程序有特定要求，则可以根据需要进行更改。点击每个文件即可查看其功能摘要。
![[Pasted image 20250805203637.png]]
#### 12.添加作者信息
在生成功能包时，Colcon需要作者信息以用于出版目的。
在Add Author Information窗格  
输入姓名和电子邮件地址。
![[Pasted image 20250805203803.png]]
#### 13.生成配置文件
最后一步——生成开始使用 MoveIt 所需的所有配置文件。  
点击 ==Browse（浏览）==按钮 ，选择一个合适的位置（例如，learnarm_ws工作区的src目录），点击 ==“创建文件夹”== ，并将其命名为 ==robot_arm_config== ，然后点击打开 。点击 ==“生成软件包”== 按钮。其会将一组启动文件和配置文件生成到您选择的目录中，如下图所示。
![[Pasted image 20250805204418.png]]
==如果点击”browse“卡住，则打开虚拟机设置-显示器配置-关闭3D图形加速==
#### 14.编译运行
==虽然 MoveIt Setup Assistant 已经生成了机械臂的配置文件，但是其生成的部分配置仍需要简单修改后才能使用。==  
编译之前修改生成的功能包robot_arm_config内的config中的==joint_limits.yaml==的最大速度和最大加速度==改为浮点数（不能是整型）==，并把has_acceleration_limits参数设置为true
```
# joint_limits.yaml allows the dynamics properties specified in the URDF to be overwritten or augmented as needed

# For beginners, we downscale velocity and acceleration limits.
# You can always specify higher scaling factors (<= 1.0) in your motion requests.  # Increase the values below to 1.0 to always move at maximum speed.
default_velocity_scaling_factor: 0.1
default_acceleration_scaling_factor: 0.1

# Specific joint properties can be changed with the keys [max_position, min_position, max_velocity, max_acceleration]
# Joint limits can be turned off with [has_velocity_limits, has_acceleration_limits]
joint_limits:
  finger1:
    has_velocity_limits: true
    max_velocity: 0.029999999999999999
    has_acceleration_limits: true
    max_acceleration: 2.0
  finger2:
    has_velocity_limits: true
    max_velocity: 0.029999999999999999
    has_acceleration_limits: true
    max_acceleration: 2.0
  joint1:
    has_velocity_limits: true
    max_velocity: 1.0
    has_acceleration_limits: true
    max_acceleration: 2.0
  joint2:
    has_velocity_limits: true
    max_velocity: 1.0
    has_acceleration_limits: true
    max_acceleration: 2.0
  joint3:
    has_velocity_limits: true
    max_velocity: 1.0
    has_acceleration_limits: true
    max_acceleration: 2.0
  joint4:
    has_velocity_limits: true
    max_velocity: 1.0
    has_acceleration_limits: true
    max_acceleration: 2.0
  joint5:
    has_velocity_limits: true
    max_velocity: 1.0
    has_acceleration_limits: true
    max_acceleration: 2.0
  joint6:
    has_velocity_limits: true
    max_velocity: 1.0
    has_acceleration_limits: true
    max_acceleration: 2.0
```
编译生成的命令：
==cd ~/dummy2_gazebo==
==colcon build --packages-select robot_arm_config  --symlink-install==
==source install/setup.bash==

MoveIt Setup Assistant生成的功能包中提供了一个测试的Launch文件`demo.launch.py`运行测试，命令如下：
```
ros2 launch panda_arm_config demo.launch.py
```
==joint_limits.yaml==的最大速度和最大加速度==改为浮点数（不能是整型）==，并把has_acceleration_limits参数设置为true,才可以正常将机械臂模型加载进来。
