```
#! /usr/bin/env python3
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import TransformStamped
from rclpy.node import ParameterType, ParameterDescriptor
from ros2_aruco_interfaces.msg import ArucoMarkers
from tf2_ros import TransformBroadcaster
典型 ROS2 Python 节点脚本头；导入所需接口和消息
```


```
class CalibrationArucoPublisher(Node):
    """Listens to ArucoMarkers and republishes the pose of one marker as a TF."""
    def __init__(self):
        super().__init__("calibration_aruco_publisher")
定义节点类，命名为 `calibration_aruco_publisher`。
```


```
tracking_base_frame_p = self.declare_parameter(
            'tracking_base_frame', value="", descriptor=ParameterDescriptor(type=ParameterType.PARAMETER_STRING))
            
self.tracking_base_frame = tracking_base_frame_p.get_parameter_value().string_value

tracking_marker_frame_p = self.declare_parameter(
            'tracking_marker_frame', value="", descriptor=ParameterDescriptor(type=ParameterType.PARAMETER_STRING))
            
self.tracking_marker_frame = tracking_marker_frame_p.get_parameter_value().string_value
self.marker_id = self.declare_parameter("marker_id", 1).get_parameter_value().integer_value

声明并读取三个参数：
- `tracking_base_frame`：TF 发布的父 frame；
    
- `tracking_marker_frame`：发布的子 frame；
    
- `marker_id`：要跟踪的 ArUco 标记 ID。
```


```
        self.tf_broadcaster = TransformBroadcaster(self)
        self.subscription = self.create_subscription(
            ArucoMarkers,
            "/aruco_markers",
            self.handle_aruco_markers,
            1)
初始化 TF 广播器，并订阅 `/aruco_markers`，缓冲深度 1。
```


```
    def handle_aruco_markers(self, msg: ArucoMarkers):
        cal_marker_pose = None
        for i, marker_id in enumerate(msg.marker_ids):
            if marker_id == self.marker_id:
                cal_marker_pose = msg.poses[i]
                break
        if cal_marker_pose is None:
            return
在回调中遍历所有检测到的 ID，找到与 `marker_id` 匹配的那一个，若未找到则跳过。
```


```
        t = TransformStamped()
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = self.tracking_base_frame
        t.child_frame_id = self.tracking_marker_frame
        t.transform.translation.x = cal_marker_pose.position.x
        t.transform.translation.y = cal_marker_pose.position.y
        t.transform.translation.z = cal_marker_pose.position.z
        t.transform.rotation = cal_marker_pose.orientation
        self.tf_broadcaster.sendTransform(t)
构造 `TransformStamped`，填充父/子 frame、时间戳和平移/旋转，然后广播到 TF 树。
```


```
def main():
    rclpy.init()
    node = CalibrationArucoPublisher()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    rclpy.shutdown()
启动与循环逻辑，同样捕获 Ctrl+C 并优雅退出。
```