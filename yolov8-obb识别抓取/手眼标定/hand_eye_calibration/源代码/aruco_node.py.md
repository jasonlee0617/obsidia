```
import rclpy
import rclpy.node
from rclpy.qos import qos_profile_sensor_data
from cv_bridge import CvBridge
import numpy as np
import cv2
import tf_transformations
from sensor_msgs.msg import CameraInfo, Image
from geometry_msgs.msg import PoseArray, Pose
from ros2_aruco_interfaces.msg import ArucoMarkers
from rcl_interfaces.msg import ParameterDescriptor, ParameterType
导入 ROS2 Python 接口、QoS 配置、OpenCV 与 NumPy，以及消息类型和参数描述工具。
```


```
class ArucoNode(rclpy.node.Node):
    def __init__(self):
        super().__init__("aruco_node")
定义节点类 `ArucoNode`，命名为 `aruco_node`。
```


```
参数声明与读取

        self.declare_parameter("marker_size", 0.0625, descriptor=ParameterDescriptor(...))
        self.declare_parameter("aruco_dictionary_id", "DICT_5X5_250", descriptor=...)
        self.declare_parameter("image_topic", "/camera/image_raw", descriptor=...)
        self.declare_parameter("camera_info_topic", "/camera/camera_info", descriptor=...)
        self.declare_parameter("camera_frame", "", descriptor=...)
        
        声明 5 个可配置参数：标记尺寸、字典 ID、图像话题、相机信息话题和可选的坐标系覆盖。

        self.marker_size = self.get_parameter("marker_size").get_parameter_value().double_value
        dictionary_id_name = self.get_parameter("aruco_dictionary_id").get_parameter_value().string_value
        image_topic = self.get_parameter("image_topic").get_parameter_value().string_value
        info_topic = self.get_parameter("camera_info_topic").get_parameter_value().string_value
        self.camera_frame = self.get_parameter("camera_frame").get_parameter_value().string_value
        self.get_logger().info(f"Marker size: {self.marker_size}")
        self.get_logger().info(f"Marker type: {dictionary_id_name}")
        self.get_logger().info(f"Image topic: {image_topic}")
        self.get_logger().info(f"Image info topic: {info_topic}")
        
        获取并存储参数值，同时打印日志以便调试。
```


```
ArUco 字典校验
        try:
            dictionary_id = cv2.aruco.__getattribute__(dictionary_id_name)
            if type(dictionary_id) != type(cv2.aruco.DICT_5X5_100):
                raise AttributeError
        except AttributeError:
            self.get_logger().error("bad aruco_dictionary_id: {}".format(dictionary_id_name))
            options = "\n".join([s for s in dir(cv2.aruco) if s.startswith("DICT")])
            self.get_logger().error("valid options: {}".format(options))
            
        动态从 `cv2.aruco` 模块获取指定字典常量，若不存在或类型不匹配，则打印可用选项并报错。
```


```
订阅与发布设置
        self.info_sub = self.create_subscription(CameraInfo, info_topic, self.info_callback, qos_profile_sensor_data)
        self.create_subscription(Image, image_topic, self.image_callback, qos_profile_sensor_data)
        self.poses_pub = self.create_publisher(PoseArray, "aruco_poses", 10)
        self.markers_pub = self.create_publisher(ArucoMarkers, "aruco_markers", 10)
        
        订阅相机内参和图像话题（均使用低延迟 QoS），并创建两个发布器：
        `aruco_poses`（仅姿态数组，便于 RViz），
        `aruco_markers`（ID＋姿态）
        
        self.info_msg = None
        self.intrinsic_mat = None
        self.distortion = None
        self.aruco_dictionary = cv2.aruco.Dictionary_get(dictionary_id)
        self.aruco_parameters = cv2.aruco.DetectorParameters_create()
        self.bridge = CvBridge()
        
        初始化存储相机参数的成员变量；获取 ArUco 检测字典和默认检测参数；初始化图像转换桥。
```


```
info_callback
    def info_callback(self, info_msg):
        self.info_msg = info_msg
        self.intrinsic_mat = np.reshape(np.array(self.info_msg.k), (3, 3))
        self.distortion = np.array(self.info_msg.d)
        self.destroy_subscription(self.info_sub)
        收到一次 `CameraInfo` 后，提取内参矩阵 `K`（3×3）和畸变系数 `d`，然后取消订阅，假设相机参数不变。
```


```
image_callback
    def image_callback(self, img_msg):
        if self.info_msg is None:
            self.get_logger().warn("No camera info has been received!")
            return
        若尚未获取相机内参，跳过处理并警告。
        
        cv_image = self.bridge.imgmsg_to_cv2(img_msg, desired_encoding="mono8")#将 ROS `Image` 转为 OpenCV 灰度图，加速后续角点检测。
        
        markers = ArucoMarkers()
        pose_array = PoseArray()
        if self.camera_frame == "":
            frame = self.info_msg.header.frame_id
        else:
            frame = self.camera_frame
        markers.header.frame_id = frame
        pose_array.header.frame_id = frame
        markers.header.stamp = img_msg.header.stamp
        pose_array.header.stamp = img_msg.header.stamp
        初始化待发布消息，设置其 header（frame_id 和 timestamp）。
        
        corners, marker_ids, rejected = cv2.aruco.detectMarkers(
            cv_image, self.aruco_dictionary, parameters=self.aruco_parameters
        )使用 OpenCV ArUco 检测器定位所有标记角点和对应 ID。
        
        if marker_ids is not None:
            rvecs, tvecs, _ = cv2.aruco.estimatePoseSingleMarkers(
                corners, self.marker_size, self.intrinsic_mat, self.distortion
            )
            若检测到标记，则基于相机内参与已知 marker 大小，计算每个标记的旋转向量 (`rvecs`) 与平移向量 (`tvecs`)。
            
        for i, marker_id in enumerate(marker_ids):
                pose = Pose()
                pose.position.x = tvecs[i][0][0]
                pose.position.y = tvecs[i][0][1]
                pose.position.z = tvecs[i][0][2]
                rot_matrix = np.eye(4)
                rot_matrix[0:3,0:3] = cv2.Rodrigues(np.array(rvecs[i][0]))[0]
                quat = tf_transformations.quaternion_from_matrix(rot_matrix)
                pose.orientation.x = quat[0]
                pose.orientation.y = quat[1]
                pose.orientation.z = quat[2]
                pose.orientation.w = quat[3]
                pose_array.poses.append(pose)
                markers.poses.append(pose)
                markers.marker_ids.append(marker_id[0])
                将平移向量直接赋给 `Pose.position`；将旋转向量经 Rodrigues→矩阵→四元数，赋给 `Pose.orientation`。
                累加到两种消息中。
                
        self.poses_pub.publish(pose_array)
        self.markers_pub.publish(markers)
        发布姿态数组与带 ID 的自定义消息。
```


```
主入口
def main():
    rclpy.init()
    node = ArucoNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
标准 ROS2 Python 节点启动/循环/销毁流程。
```