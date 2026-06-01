```
import rclpy
from rclpy.node import Node
import cv2
import numpy as np
from cv_bridge import CvBridge
from sensor_msgs.msg import Image, CameraInfo
- 指定使用 Python 3 解释器执行脚本
- 导入 ROS 2 核心库和节点基类
- 导入 OpenCV（计算机视觉库）和 NumPy（数值计算库）
- 导入 ROS 与 OpenCV 图像转换工具
- 导入 ROS 图像消息和相机参数消息类型
```

```
class ArucoPoseEstimator(Node):
    """Node to estimate the pose of ArUco markers..."""
- 定义 ArUco 姿态估计节点类
```

```
    def __init__(self):
        super().__init__('aruco_pose_estimator')
- 节点初始化，设置节点名称为 `aruco_pose_estimator`
```

```
        self.bridge = CvBridge()
- 创建 CvBridge 实例用于 ROS-OpenCV 图像转换
```

```
        self.image_subscription = self.create_subscription(
            Image,
            '/camera/camera/color/image_raw',
            self.image_callback,
            10)
- 创建图像订阅器，订阅原始 RGB 图像话题
```

```
        self.camera_info_subscription = self.create_subscription(
            CameraInfo,
            '/camera/camera/color/camera_info',
            self.camera_info_callback,
            10)
        self.camera_info_received = False
- 创建相机参数订阅器，订阅相机内参话题
- 设置相机参数接收标志
```

```
        self.camera_matrix = None
        self.dist_coeffs = None
- 初始化相机内参矩阵和畸变系数
```

```
        self.aruco_dict = cv2.aruco.getPredefinedDictionary(
            cv2.aruco.DICT_5X5_250)
        self.aruco_params = cv2.aruco.DetectorParameters()
- 选择 ArUco 标记字典（5x5，250种标记）
- 创建 ArUco 检测参数对象
```

```
        self.image_publisher = self.create_publisher(Image, '/aruco_image', 10)
- 创建图像发布器，用于发布带检测结果的图像
```

```
    def camera_info_callback(self, msg):
        K = np.array(msg.k).reshape(3, 3)
        D = np.array(msg.d)
        self.camera_matrix = K
        self.dist_coeffs = D
        self.camera_info_received = True
        self.destroy_subscription(self.camera_info_subscription)

- 相机参数回调函数：
    - 提取内参矩阵并重塑为 3x3
    - 提取畸变系数
    - 设置接收标志
    - 取消订阅（只需获取一次参数）
```

```
    def image_callback(self, msg):
        if not self.camera_info_received:
            return
- 图像回调函数，首先检查是否已获取相机参数
```

```
        cv_image = self.bridge.imgmsg_to_cv2(msg, "bgr8")
- 将 ROS 图像消息转换为 OpenCV BGR 格式
```

```
        corners, ids, rejectedImgPoints = cv2.aruco.detectMarkers(
            cv_image, self.aruco_dict)
- 检测图像中的 ArUco 标记
```

```
        if ids is not None:
            rvecs, tvecs, _ = cv2.aruco.estimatePoseSingleMarkers(
                corners, 0.1, self.camera_matrix, self.dist_coeffs)
- 如果检测到标记：
    - 估计每个标记的 3D 姿态（旋转向量和平移向量）
    - 假设标记尺寸为 0.1 米
```

```
            for i in range(len(ids)):
                cv2.aruco.drawDetectedMarkers(cv_image, corners, ids)
                cv2.drawFrameAxes(cv_image, self.camera_matrix,
                                  self.dist_coeffs, rvecs[i], tvecs[i], 0.1)
- 在图像上绘制：
    - 标记边界框和 ID
    - 3D 坐标系轴（长度 0.1 米）
```

```
        overlay_msg = self.bridge.cv2_to_imgmsg(cv_image, "bgr8")
        self.image_publisher.publish(overlay_msg)
- 将处理后的图像转换回 ROS 消息并发布
```

```
def main(args=None):
    rclpy.init(args=args)
    aruco_pose_estimator = ArucoPoseEstimator()
    rclpy.spin(aruco_pose_estimator)
    aruco_pose_estimator.destroy_node()
    rclpy.shutdown()

- 主函数：
    - 初始化 ROS
    - 创建节点实例
    - 启动节点循环
     
    - 清理资源
```

```
if __name__ == '__main__':
    main()
- 脚本入口点
```

### 模块化功能与联系

#### 1. **输入模块**
    - 图像订阅：获取实时相机图像
    - 相机参数订阅：获取相机内参和畸变系数
    - 依赖关系：姿态估计需要相机参数
#### 2. **核心处理模块**
    - **ArUco检测**：
        - 使用 OpenCV 检测图像中的 ArUco 标记
        - 输出标记角点位置和 ID
    - **姿态估计**：
        - 利用相机参数和标记物理尺寸
        - 计算标记相对于相机的 3D 位置（平移向量）和方向（旋转向量）
        - 基于透视 n 点（PnP）算法
#### 3. **可视化模块**
    - 绘制标记边界框和 ID
    - 绘制 3D 坐标系轴显示姿态
    - 生成带标注的输出图像
#### 4. **输出模块**
    - 发布带检测结果的可视化图像
    - 通过 `/aruco_image` 话题输出

### 实现功能总结

#### 1. **ArUco 标记检测**
    - 实时识别图像中的预定义 ArUco 标记
    - 获取标记的精确角点位置和唯一 ID
#### 2. **6-DoF 姿态估计**
    - 计算标记在相机坐标系中的：
        - 三维位置 (x, y, z)
        - 三维方向（旋转向量）
    - 精度取决于：
        - 相机标定质量
        - 标记物理尺寸准确性
        - 图像分辨率
#### 3. **实时可视化**
    - 在原始图像上叠加：
        - 标记边界框（绿色）
        - 标记 ID 文本
        - 3D 坐标系轴（X:红, Y:绿, Z:蓝）
#### 4. **ROS 集成**
    - 标准 ROS 2 节点架构
    - 话题通信机制
    - 与相机驱动解耦
    - 提供可视化输出接口

功能实现逻辑流程
![[deepseek_mermaid_20250724_59b692.svg]]