```
import os
from launch import LaunchDescription
from launch.actions import ExecuteProcess
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
from moveit_configs_utils import MoveItConfigsBuilder


def generate_launch_description():
    # planning_context
    moveit_config = (
        MoveItConfigsBuilder("dummy2-gripperv2",package_name="dummy2-gripperv2_moveit_config")
        .robot_description(file_path="config/dummy2-gripperv2.urdf.xacro")
        .trajectory_execution(file_path="config/moveit_controllers.yaml")
        .robot_description_kinematics(file_path="config/kinematics.yaml")
        .planning_pipelines(
            pipelines=["ompl"]
        )
        .to_moveit_configs()
    )

    # Load  ExecuteTaskSolutionCapability so we can execute found solutions in simulation
    move_group_capabilities = {
        "capabilities": "move_group/ExecuteTaskSolutionCapability"
    }

    # Start the actual move_group node/action server
    run_move_group_node = Node(
        package="moveit_ros_move_group",
        executable="move_group",
        output="screen",
        parameters=[
            moveit_config.to_dict(),
            move_group_capabilities,
        ],
    )

    # RViz
    rviz_config_file = (
        get_package_share_directory("mtc_tutorial") + "/launch/mtc.rviz"
    )
    rviz_node = Node(
        package="rviz2",
        executable="rviz2",
        output="log",
        arguments=["-d", rviz_config_file],
        parameters=[
            moveit_config.robot_description,
            moveit_config.robot_description_semantic,
            moveit_config.robot_description_kinematics,
        ],
    )

    # Static TF
    static_tf = Node(
        package="tf2_ros",
        executable="static_transform_publisher",
        name="static_transform_publisher",
        output="log",
        arguments=["--frame-id", "world", "--child-frame-id", "base_link"],
    )

    # Publish TF
    robot_state_publisher = Node(
        package="robot_state_publisher",
        executable="robot_state_publisher",
        name="robot_state_publisher",
        output="both",
        parameters=[
            moveit_config.robot_description,
        ],
    )

    # ros2_control using FakeSystem as hardware
    ros2_controllers_path = os.path.join(
        get_package_share_directory("dummy2-gripperv2_moveit_config"),
        "config",
        "ros2_controllers.yaml",
    )
    ros2_control_node = Node(
        package="controller_manager",
        executable="ros2_control_node",
        parameters=[ros2_controllers_path],
        remappings=[
            ("/controller_manager/robot_description", "/robot_description"),
        ],
        output="both",
    )

    # Load controllers
    load_controllers = []
    for controller in [
        "dummy2_arm_controller",
        "hand_controller",
        "joint_state_broadcaster",
    ]:
        load_controllers += [
            ExecuteProcess(
                cmd=["ros2 run controller_manager spawner {}".format(controller)],
                shell=True,
                output="screen",
            )
        ]

    return LaunchDescription(
        [
            rviz_node,
            static_tf,
            robot_state_publisher,
            run_move_group_node,
            ros2_control_node,
        ]
        + load_controllers
    ) 
```

### 1.**导入模块**：
```
import os
from launch import LaunchDescription
from launch.actions import ExecuteProcess
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
from moveit_configs_utils import MoveItConfigsBuilder

- `os`：操作系统接口（文件路径处理）
- `LaunchDescription`：ROS 2启动文件的核心容器
- `ExecuteProcess`：执行外部命令/进程
- `Node`：声明ROS节点
- `get_package_share_directory`：获取ROS包的安装路径
- `MoveItConfigsBuilder`：MoveIt配置生成工具
```

### 2.**构建MoveIt配置**：
```
def generate_launch_description():
    # planning_context
    moveit_config = (
        MoveItConfigsBuilder("dummy2-gripperv2",package_name="dummy2-gripperv2_moveit_config")
        .robot_description(file_path="config/dummy2-gripperv2.urdf.xacro")
        .trajectory_execution(file_path="config/moveit_controllers.yaml")
        .robot_description_kinematics(file_path="config/kinematics.yaml")
        .planning_pipelines(pipelines=["ompl"])
        .to_moveit_configs()
    )

- **机器人名称**：`dummy2-gripperv2`
- **配置包名**：`dummy2-gripperv2_moveit_config`
- **URDF文件**：`config/dummy2-gripperv2.urdf.xacro`（机器人模型）
- **控制器配置**：`config/moveit_controllers.yaml`（轨迹执行参数）
- **运动学配置**：`config/kinematics.yaml`（逆运动学求解器参数）
- **规划管道**：启用OMPL规划器（默认规划算法）
- **最终转换**：`.to_moveit_configs()`生成完整配置字典
```

### 3.**配置MoveGroup能力**：
```
    move_group_capabilities = {
        "capabilities": "move_group/ExecuteTaskSolutionCapability"
    }
启用`ExecuteTaskSolutionCapability`：允许执行MoveIt Task Constructor生成的规划方案
```

### 4.**启动MoveGroup主节点**：
```
    run_move_group_node = Node(
        package="moveit_ros_move_group",
        executable="move_group",
        output="screen",
        parameters=[moveit_config.to_dict(), move_group_capabilities],
    )

- **功能**：MoveIt的核心协调节点（规划/执行/场景管理）
- **输出**：控制台打印（`output="screen"`）
- **参数**：合并MoveIt配置和能力扩展
```

### 5.**启动RViz可视化**：
```
    rviz_config_file = os.path.join(
        get_package_share_directory("mtc_tutorial"), "launch/mtc.rviz"
    )
    rviz_node = Node(
        package="rviz2",
        executable="rviz2",
        output="log",
        arguments=["-d", rviz_config_file],
        parameters=[...],
	    )

- **配置文件**：`mtc_tutorial`包中的`mtc.rviz`（预置的MoveIt可视化配置）
- **参数注入**：
    - 机器人描述（URDF）
    - 语义描述（SRDF）
    - 运动学参数
- **输出**：日志文件（`output="log"`）
```

### 6.**静态TF发布**：
```
    static_tf = Node(
        package="tf2_ros",
        executable="static_transform_publisher",
        arguments=["--frame-id", "world", "--child-frame-id", "base_link"],
    )
- 建立`world` → `base_link`坐标系变换
- 固定机器人基座标系与世界坐标系的相对位置
```

### 7.**机器人状态发布**：
```
    robot_state_publisher = Node(
        package="robot_state_publisher",
        executable="robot_state_publisher",
        parameters=[moveit_config.robot_description],
    )
- 解析URDF并发布`/robot_description`话题
- 发布关节状态→TF变换（动态）
```

### 8.**ros2_control仿真节点**：
```
    ros2_controllers_path = os.path.join(
        get_package_share_directory("dummy2-gripperv2_moveit_config"),
        "config/ros2_controllers.yaml",
    )
    ros2_control_node = Node(
        package="controller_manager",
        executable="ros2_control_node",
        parameters=[ros2_controllers_path],
        remappings=[("/controller_manager/robot_description", "/robot_description")],
	    )

- **配置文件**：`ros2_controllers.yaml`（定义关节控制器）
- **重映射**：将控制器需要的`/robot_description`指向标准话题
- **功能**：用FakeSystem模拟硬件接口
```

### 9.**动态加载控制器**：
```
    load_controllers = []
    for controller in ["dummy2_arm_controller", "hand_controller", "joint_state_broadcaster"]:
        load_controllers.append(ExecuteProcess(
            cmd=["ros2 run controller_manager spawner {}".format(controller)],
            shell=True,
            output="screen",
        ))

- **手臂控制器**：`dummy2_arm_controller`（执行轨迹）
- **手爪控制器**：`hand_controller`（夹爪控制）
- **关节状态广播器**：发布`sensor_msgs/JointState`
- **执行方式**：通过`spawner`脚本动态加载
```

### 10.**组装启动描述**：
```
    return LaunchDescription([
        rviz_node,
        static_tf,
        robot_state_publisher,
        run_move_group_node,
        ros2_control_node,
    ] + load_controllers)

- **节点顺序**：
    1. RViz可视化
    2. 静态TF发布
    3. 机器人状态发布
    4. MoveGroup主节点
    5. ros2_control节点
- **动态加载**：控制器按列表顺序启动
```

### 核心功能流程图
![[Pasted image 20250804010026.png]]
