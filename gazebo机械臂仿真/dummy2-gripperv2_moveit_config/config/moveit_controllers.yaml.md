```
# MoveIt uses this configuration for controller management
moveit_controller_manager: moveit_simple_controller_manager/MoveItSimpleControllerManager

moveit_simple_controller_manager:
  controller_names:
    - dummy2_arm_controller
    - hand_controller

  dummy2_arm_controller:
    type: FollowJointTrajectory
    action_ns: follow_joint_trajectory
    default: true
    joints:
      - joint1
      - joint2
      - joint3
      - joint4
      - joint5
      - joint6
    action_ns: follow_joint_trajectory
    default: true

  hand_controller:
    type: GripperCommand
    joints:
      - figer1
      - figer2
    action_ns: gripper_cmd
    default: true
```

#### 1.控制器管理器类型
=MoveItSimpleControllerManager== 是 MoveIt! 提供的默认控制器管理器(==负责连接 MoveIt! 规划结果与实际的 ROS 控制器==)
#### 2.控制器管理器配置
moveit_simple_controller_manager( 开始控制器管理器的具体配置块)
#### 3.控制器名称列表:
  controller_names:
    - dummy2_arm_controller
    - hand_controller
- **核心配置**：声明 MoveIt! 将管理的控制器
- **dummy2_arm_controller**：机械臂轨迹跟踪控制器
- **hand_controller**：夹爪抓取控制器
#### 4.机械臂控制器详细配置
  dummy2_arm_controller:
    type: FollowJointTrajectory
- **控制器类型**：`FollowJointTrajectory`
- **对应接口**：遵循 `control_msgs/action/FollowJointTrajectory` 动作接口
- **功能**：执行关节空间轨迹跟踪

action_ns: follow_joint_trajectory
- **动作命名空间**：控制器监听的动作名称
- **完整路径**：`/dummy2_arm_controller/follow_joint_trajectory`
- **通信协议**：ROS 2 Action

default: true
- **默认控制器**：当 MoveIt! 需要控制机械臂时首选此控制器
- **优先级**：允许多控制器配置时指定主控制器

 joints:
      - joint1
      - joint2
      - joint3
      - joint4
      - joint5
      - joint6
- **关节列表**：定义控制器管理的 6 个关节
- **匹配要求**：必须与 URDF 和 `ros2_control.yaml` 中的关节名一致
#### 5.夹爪控制器详细配置
  hand_controller:
    type: GripperCommand
- **控制器类型**：`GripperCommand`
- **对应接口**：遵循 `control_msgs/action/GripperCommand` 动作接口
- **专用功能**：控制夹爪开合动作

joints:
      - figer1
      - figer2
- **关节列表**：控制两个夹爪关节
- **关节数要求**：夹爪控制器通常控制对称的 2 个关节

action_ns: gripper_cmd
    default: true
- **动作命名空间**：`/hand_controller/gripper_cmd`
- - **默认控制器**：夹爪操作的默认控制器

![[Pasted image 20250802221737.png]]

### 关键配置项总结
|配置项|机械臂控制器|夹爪控制器|作用|
|---|---|---|---|
|**类型**|`FollowJointTrajectory`|`GripperCommand`|接口协议|
|**动作空间**|`follow_joint_trajectory`|`gripper_cmd`|动作地址|
|**默认**|`true`|`true`|首选标志|
|**关节**|joint1-joint6|figer1, figer2|控制对象|
