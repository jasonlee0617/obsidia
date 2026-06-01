```
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

### 控制器管理器全局配置
```
controller_manager:
  ros__parameters:
    update_rate: 100  # Hz

- **`controller_manager`**: ROS 2 控制系统的核心组件
- **`update_rate`**: 控制循环频率设为 100Hz (10ms周期)
    - 影响所有控制器的执行频率
    - 典型值范围：50-1000Hz，取决于硬件性能
```

### 控制器类型声明
```
    dummy2_arm_controller:
      type: joint_trajectory_controller/JointTrajectoryController

    hand_controller:
      type: position_controllers/GripperActionController

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcasterv

1. **机械臂控制器**:
    - `dummy2_arm_controller`: 自定义名称
    - `type`: 使用标准关节轨迹控制器
    - 功能：执行预定义的关节轨迹（常用于机械臂）

2. **手爪控制器**:
    - `hand_controller`: 自定义名称
    - `type`: 专用的夹爪动作控制器
    - 功能：控制夹爪的开合动作

3. **关节状态广播器**:
    - `joint_state_broadcaster`: 标准名称
    - `type`: 关节状态广播组件
    - 功能：将硬件接口数据发布到 `/joint_states` 话题
```

### 机械臂控制器详细配置
```
dummy2_arm_controller:
  ros__parameters:
    joints:
      - joint1
      - joint2
      - joint3
      - joint4
      - joint5
      - joint6

- **关节列表**：指定控制的6个关节 (joint1-joint6)
- 命名应符合URDF中的关节定义
```

```
    command_interfaces:
      - positionv

- **命令接口**：只使用位置控制模式
- 可选值：`position`/`velocity`/`effort`
- 表示控制器向硬件发送位置指令
```

```
    state_interfaces:
      - position
      - velocity

- **状态接口**：读取位置和速度反馈
- 硬件必须提供这些数据源
- 速度信息可用于控制算法（如PID
```

### 手爪控制器详细配置
```
hand_controller:
  ros__parameters:
    joints:
      - figer1
      - figer2

- **关节列表**：控制两个夹爪关节 (figer1, figer2)
- **注意**：`figer` 可能是拼写错误，应为 `finger`
- 未显式声明接口，使用控制器默认配置：
    - `GripperActionController` 默认使用位置接口
```

### 配置关键点总结
#### 制器类型选择

![[Pasted image 20250802211549.png]]
### 系统工作流程
#### 1.启动时：
```
from controller_manager import spawner
spawner.spawn(
    'controller_manager',
    'dummy2_arm_controller',
    'hand_controller',
    'joint_state_broadcaster'
)
```
#### 2.行时数据流：
```
[MoveIt] → 轨迹命令 → dummy2_arm_controller → 硬件接口
[Gripper Action] → 目标位置 → hand_controller → 硬件接口
硬件反馈 → joint_state_broadcaster → /joint_states
```

#### 3.典型应用场景：
- 机械臂执行笛卡尔空间轨迹
- 夹爪同步抓取物体
- 实时监控所有关节状态