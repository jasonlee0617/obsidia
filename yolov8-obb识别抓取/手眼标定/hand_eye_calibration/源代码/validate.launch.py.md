```
import os
import yaml
from ament_index_python.packages import get_package_share_directory
from launch_ros.actions import Node
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration

导入操作系统接口模块，用于文件路径操作。
导入YAML处理模块，用于解析配置文件。
导入ROS 2功能包路径获取工具。
导入ROS节点启动工具。
导入启动描述构建基类。
导入启动参数声明和子启动文件包含工具
导入Python启动文件源。
导入启动参数配置工具。
```

```
def load_yaml(package_name, file_name):
    package_path = get_package_share_directory(package_name)
    absolute_file_path = os.path.join(package_path, file_name)
    with open(absolute_file_path, "r", encoding="utf-8") as file:
        return yaml.safe_load(file)
        
    定义YAML配置文件加载函数，获取功能包路径并解析YAML文件。
```

```
def generate_launch_description():
定义生成启动描述的主函数。
```

```
    ar_model_config = LaunchConfiguration("ar_model")
    ar_model_arg = DeclareLaunchArgument(
        "ar_model",
        default_value="mk1",
        choices=["mk1", "mk2", "mk3"],
        description="Model of dummy2",
        
    声明启动参数 `ar_model`，用于选择机器人模型版本（mk1/mk2/mk3）。
    )
```

```
    realsense = IncludeLaunchDescription(
        PythonLaunchDescriptionSource([
            os.path.join(
                get_package_share_directory("realsense2_camera"),
                "launch",
                "rs_launch.py",
            )
        ])
    )
    
启动Realsense相机节点，包含官方启动文件。
```

```
    aruco_params = os.path.join(
        get_package_share_directory("dummy2_hand_eye_calibration"),
        "config",
        "aruco_parameters.yaml",
    )
    aruco_recognition_node = Node(
        package="ros2_aruco", 
        executable="aruco_node", 
        parameters=[aruco_params]
    )
配置并启动Aruco标记识别节点，加载专用参数文件。
```

```
    hand_eye_tf_publisher = Node(
        package="dummy2_hand_eye_calibration",
        executable="handeye_publisher.py",
        name="handeye_publisher",
        parameters=[{"calibration_name": "dummy2_calibration"}],
    )
    
启动手眼标定TF发布节点，加载预存的标定数据。
```

```
    follow_aruco_node = Node(
        package="dummy2_hand_eye_calibration",
        executable="follow_aruco_marker.py",
        name="follow_aruco_marker",
        output="screen",
    )
    启动Aruco标记跟随控制节点，输出显示在终端。
```

```
    ar_moveit_launch = PythonLaunchDescriptionSource([
        os.path.join(
            get_package_share_directory("dummy2-gripperv3_moveit_config"),
            "launch",
            "demo.launch.py",
        )
    ])
    rviz_config_file = os.path.join(
        get_package_share_directory("dummy2_hand_eye_calibration"), 
        "rviz", 
        "validate.rviz"
    )
    ar_moveit_args = {
        "include_gripper": "False",
        "rviz_config_file": rviz_config_file,
        "ar_model_config": ar_model_config,
    }.items()
    ar_moveit = IncludeLaunchDescription(
        ar_moveit_launch, 
        launch_arguments=ar_moveit_args
    )
配置并启动MoveIt!运动规划系统：
1. 指定MoveIt!启动文件
2. 设置RViz配置文件路径
3. 传递参数（禁用夹爪/自定义RViz配置/机器人型号）
```

```
    return LaunchDescription([
        ar_model_arg,
        realsense,
        hand_eye_tf_publisher,
        aruco_recognition_node,
        follow_aruco_node,
        ar_moveit,
    ])
构建并返回完整的启动描述。
```

![[Pasted image 20250725001142.png]]

![[deepseek_mermaid_20250724_f6d5a8.svg]]