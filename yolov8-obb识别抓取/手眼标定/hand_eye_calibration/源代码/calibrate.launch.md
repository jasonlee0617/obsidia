```
import os
from ament_index_python.packages import get_package_share_directory
from launch_ros.actions import Node
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource

`import os`：用于处理文件路径拼接。
`get_package_share_directory`：查找已安装 ROS2 包的 share 目录，方便定位资源。
`Node`：启动 ROS2 节点的接口。 
`LaunchDescription`：封装整个 launch 配置。
`IncludeLaunchDescription`／`PythonLaunchDescriptionSource`：允许在一个 launch 文件中包含并执行另一个 Python launch。
```


```
def generate_launch_description():
定义标准的 ROS2 launch 入口函数，返回一个 `LaunchDescription` 实例。
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
包含并启动 `realsense2_camera` 包中的 `rs_launch.py`，以启动 RealSense 相机驱动并发布图像、相机信息等话题。
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
        "moveit_with_camera.rviz",
    )
    ar_moveit_args = {
        "include_gripper": "False",
        "rviz_config_file": rviz_config_file,
    }.items()
    ar_moveit = IncludeLaunchDescription(ar_moveit_launch, launch_arguments=ar_moveit_args)
    
加载机器人 MoveIt 配置（`demo.launch.py`），并传入参数：
`include_gripper`: 是否可视化夹爪，设为 `False`。
`rviz_config_file`: 指定 RViz 界面配置文件，用于同时显示机器人和相机视角。
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
    
- 拼接用户包中 ArUco 参数文件路径（包含 marker 尺寸、字典 ID 等）。
- 启动前文 `aruco_node.py`，加载该参数配置，开始图像中 ArUco 标记识别与姿态发布。
```


```
    calibration_args = {
        "name": "dummy2_calibration",
        "calibration_type": "eye_on_base",
        "robot_base_frame": "base_link",
        "robot_effector_frame": "link6_1",
        "tracking_base_frame": "camera_color_optical_frame",
        "tracking_marker_frame": "calibration_aruco",
    }
    
定义手眼标定核心参数：
- `calibration_type`: “eye_on_base” 表示相机挂在机器人基座上（而非手臂末端）。
- 其余指定各个 TF frame 的名称。
```


```
    calibration_aruco_publisher = Node(
        package="dummy2_hand_eye_calibration",
        executable="calibration_aruco_publisher.py",
        name="calibration_aruco_publisher",
        output="screen",
        parameters=[{
            "tracking_base_frame": calibration_args["tracking_base_frame"],
            "tracking_marker_frame": calibration_args["tracking_marker_frame"],
            "marker_id": 1,
        }],
    )
启动 `calibration_aruco_publisher.py` 节点，订阅 `/aruco_markers` 话题，筛选出 ID 为 1 的标记，并将其位姿以 TF 形式发布，供标定算法使用。
```


```
    easy_handeye2 = IncludeLaunchDescription(
        PythonLaunchDescriptionSource([
            os.path.join(
                get_package_share_directory("easy_handeye2"),
                "launch",
                "calibrate.launch.py",
            )
        ]),
        launch_arguments=calibration_args.items(),
    )
包含并启动 `easy_handeye2` 包自带的标定流程 launch，将上述 `calibration_args` 传入，启动 handeye 服务与 GUI。
```


```
    static_tf_publisher = Node(
        package="tf2_ros",
        executable="static_transform_publisher",
        arguments=["0", "0", "0", "0", "0", "0", "world", "camera_link"],
        output="screen",
    )
发布一个静态 TF 变换，将 `world` 坐标系和 `camera_link` 坐标系以单位变换（零平移、单位四元数）关联，确保 TF 树连通。
```


```
    ld = LaunchDescription()
    ld.add_action(realsense)
    ld.add_action(static_tf_publisher)
    ld.add_action(ar_moveit)
    ld.add_action(aruco_recognition_node)
    ld.add_action(calibration_aruco_publisher)
    ld.add_action(easy_handeye2)
    return ld
将所有上述节点和包含动作按顺序加入 `LaunchDescription` 并返回，启动时依次执行。
```

流程工作图
![[Pasted image 20250710211746.png]]
![[deepseek_mermaid_20250608_a04913.png]]