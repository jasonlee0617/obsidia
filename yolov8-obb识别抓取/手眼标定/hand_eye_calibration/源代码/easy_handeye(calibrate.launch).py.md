
```
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.conditions import IfCondition
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node
from easy_handeye2.common_launch import arg_calibration_type, arg_tracking_base_frame, arg_tracking_marker_frame, arg_robot_base_frame, arg_robot_effector_frame
导入 launch 核心 API 以及 `easy_handeye2` 提供的标准参数声明。
```


```def generate_launch_description():
    arg_name = DeclareLaunchArgument('name')
    calibration_type = LaunchConfiguration('calibration_type')
声明 `name` 参数，用于标定实例命名；获取 `calibration_type` 配置。
```


```
    node_dummy_calib_eih = Node(..., condition=IfCondition('False'), arguments=[...,'--frame-id', LaunchConfiguration('robot_effector_frame'), '--child-frame-id', LaunchConfiguration('tracking_base_frame')])
    node_dummy_calib_eob = Node(..., condition=IfCondition('True'), arguments=[...,'--frame-id', LaunchConfiguration('robot_base_frame'), '--child-frame-id', LaunchConfiguration('tracking_base_frame')])
    
根据 `calibration_type`（eye on hand/base）选择性地启动一个 dummy static transform publisher，替代真实的 effector→camera 或 base→camera TF。
```


```
    handeye_server = Node(
        package='easy_handeye2',
        executable='handeye_server',
        name='handeye_server',
        parameters=[{ ... }]
    )
    handeye_rqt_calibrator = Node(
        package='easy_handeye2',
        executable='rqt_calibrator.py',
        name='handeye_rqt_calibrator',
        parameters=[{ ... }]
    )

- 启动核心标定服务 `handeye_server`，它接收从 `/aruco_markers`（via TF）和 robot state 获取的对应 TF，对点对齐采样并计算手眼变换。
- 启动 `rqt_calibrator` GUI，用于交互式触发采样、查看误差并保存结果。
```


```
    return LaunchDescription([
        arg_name,
        arg_calibration_type, arg_tracking_base_frame, arg_tracking_marker_frame, arg_robot_base_frame, arg_robot_effector_frame,
        node_dummy_calib_eih, node_dummy_calib_eob,
        handeye_server, handeye_rqt_calibrator,
    ])
将参数声明与节点动作打包并返回，供父级 launch（`calibrate.launch.py`）包含。
```