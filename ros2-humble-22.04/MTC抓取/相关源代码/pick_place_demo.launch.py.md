```
from launch import LaunchDescription
from launch_ros.actions import Node
from moveit_configs_utils import MoveItConfigsBuilder

def generate_launch_description():
    moveit_config = MoveItConfigsBuilder("dummy2-gripperv2").to_dict()

    # MTC Demo node
    pick_place_demo = Node(
        package="mtc_tutorial",
        executable="mtc_node",
        output="screen",
        parameters=[
            moveit_config,
        ],
    )

    return LaunchDescription([pick_place_demo])
```

### 1.**导入模块**：
```
from launch import LaunchDescription
from launch_ros.actions import Node
from moveit_configs_utils import MoveItConfigsBuilder

- `LaunchDescription`：ROS 2启动描述的核心类
- `Node`：用于声明ROS节点
- `MoveItConfigsBuilder`：MoveIt配置构建工具（简化配置加载）
```

### 2.**启动描述入口函数**：
```
def generate_launch_description():
- ROS 2启动文件的固定格式
- 必须返回`LaunchDescription`对象
```

### 3.**一键式MoveIt配置构建**：
```
    moveit_config = MoveItConfigsBuilder("dummy2-gripperv2").to_dict()

- **机器人名称**：`dummy2-gripperv2`
- **自动推导**：
    - 默认包名：`dummy2-gripperv2_moveit_config`
    - 自动加载标准配置文件：
        - `config/dummy2-gripperv2.urdf.xacro` (URDF模型)
        - `config/moveit_controllers.yaml` (控制器配置)
        - `config/kinematics.yaml` (运动学参数)
- **配置转换**：`.to_dict()`将配置转为字典格式
- **规划管道**：默认启用OMPL规划器
```

### 4.**MTC演示节点配置**：
```
    # MTC Demo node
    pick_place_demo = Node(
        package="mtc_tutorial",
        executable="mtc_node",
        output="screen",
        parameters=[
            moveit_config,
        ],
    )

- **功能包**：`mtc_tutorial` (包含MoveIt Task Constructor示例)
- **可执行文件**：`mtc_node` (实现拾取放置任务的核心节点)
- **输出设置**：
    - `output="screen"`：控制台实时输出
    - 便于调试任务执行过程
- **参数注入**：
    - 将完整的MoveIt配置字典传入节点
    - 包含机器人模型/运动学/控制器等关键信息
```

### 5.**构建启动描述**：
```
    return LaunchDescription([pick_place_demo])
- 创建`LaunchDescription`对象
- 仅包含`mtc_node`单个节点
- 精简设计（依赖其他launch文件启动基础组件）
```

![[Pasted image 20250804010634.png]]
