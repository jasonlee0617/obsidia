
### 1.安装 ultralytics
```
pip install ultralytics opencv-python
```
### 2.安装相关依赖
#### 1.安装环境
```
Python: 3.10.12 (main, Aug 15 2025, 14:32:43) [GCC 11.4.0]
NumPy:  1.23.0
OpenCV: 4.12.0
transforms3d: 0.4.2
```

#### 2.安装相关依赖
```
sudo apt-get install ros-humble-rqt-image-view
sudo apt install ros-humble-rqt ros-humble-rqt-common-plugins
```
### 3.yolov8-obb的ros2简单部署
#### 1.创建ROS2 包 ==`yolov8_grasping`==
```
cd ~/S622_robotarm/src
ros2 pkg create --build-type ament_python yolov8_grasping --dependencies rclpy sensor_msgs cv_bridge depthai_ros_driver
```
#### 2.修改setup.py文件
##### 1.添加引用
```
import os
from glob import glob
```
##### 2.添加data_files
```
data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
    ],
```
##### 3.添加节点
```
    entry_points={
        'console_scripts': [
            'yolo_detector_obb = yolov8_grasping.yolo_detector_obb_node:main'
        ],
    },
```
#### 2.编写 ROS2 YOLOv8 节点
##### 1.进入 `yolov8_grasping` 包目录：
```
cd ~/S622_robotarm/src/yolov8_grasping
```

##### 2.创建`yolo_detector_obb_node.py`

在 `yolov8_grasping/yolov8_grasping/` 下创建一个新的 Python 文件 yolo_detector_obb_node.py：
```
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node

from sensor_msgs.msg import Image, CameraInfo
from geometry_msgs.msg import PointStamped
from std_msgs.msg import Header, Float32MultiArray

from cv_bridge import CvBridge, CvBridgeError
from ultralytics import YOLO

import cv2
import numpy as np
import threading
import math


# -----------------------------
# Angle helpers
# -----------------------------

def wrap_to_pi(a: float) -> float:  # 包装后的角度，范围[-π, π]
    return float((a + math.pi) % (2.0 * math.pi) - math.pi)


def angle_diff(a: float, b: float) -> float:  #计算两个角度之间的最小差值
    return wrap_to_pi(a - b)


def choose_equivalent_angle(cur: float, prev: float, period: float) -> float:
    """
    参数：
    cur: 当前测量角度（已包装到[-π, π]）
    prev: 前一个角度
    period: 周期（pen: π, box/cube: π/2）
    
    返回：
    最接近prev的等价角度
    """

    best = cur
    best_err = abs(angle_diff(cur, prev))
    # 尝试多个周期偏移（-4到4个周期）
    for k in (-4, -3, -2, -1, 0, 1, 2, 3, 4):
        cand = cur + k * period # 生成候选角度
        err = abs(angle_diff(cand, prev)) # 计算与prev的差值
        if err < best_err:
            best_err = err
            best = cand
    return wrap_to_pi(best)


def yaw_0_to_pi_right0_left180(corners_2d: np.ndarray) -> float:
    """
    从OBB角点计算偏航角（范围：0到π）
    
    物理意义：
    0°: 物体右侧朝右（最长的边水平向右）
    90°: 物体右侧朝下
    180°: 物体右侧朝左
    
    参数：
    corners_2d: 4×2数组，OBB的四个角点坐标
    
    返回：
    偏航角（弧度），范围[0, π]
    """
    c = corners_2d.astype(np.float32)
    best_v = None
    best_len = -1.0
    # 遍历四条边，找出最长的边
    for i in range(4):
        v = c[(i + 1) % 4] - c[i]  # 计算第i条边向量：从角点i到角点(i+1)%4
        L = float(np.linalg.norm(v)) # 计算边的长度（L2范数）
        # 更新最长边
        if L > best_len:
            best_len = L
            best_v = v

    if best_v is None or best_len < 1e-6:
        return 0.0

    dx, dy = float(best_v[0]), float(best_v[1])    # 提取边向量的x,y分量
    yaw = math.atan2(abs(dy), dx)  # [0, pi]， 计算角度：使用atan2(|dy|, dx)将角度限制在[0, π]
    yaw = max(0.0, min(math.pi, yaw)) # 确保角度在[0, π]范围内
    return float(yaw)


# -----------------------------
# OBB extraction
# -----------------------------

def try_extract_obb_corners(result, i_det):
    obb = getattr(result, 'obb', None)
    if obb is None:
        return None
    if hasattr(obb, 'xyxyxyxy') and obb.xyxyxyxy is not None and len(obb.xyxyxyxy) > i_det:
        try:
            arr = obb.xyxyxyxy[i_det].detach().cpu().numpy().reshape(-1, 2)
            if arr.shape[0] >= 4:
                return arr[:4]
        except Exception:
            pass
    return None


# -----------------------------
# Node
# -----------------------------

class YoloDetectorNode(Node):
    def __init__(self):
        super().__init__('yolov8_detector_yaw_0_180')

        # Params
        self.declare_parameter('model_path', '/home/jasonlee/S622_robotarm/yolo-obb3.pt')
        self.declare_parameter('device', 'cpu')
        self.declare_parameter('conf', 0.5)
        self.declare_parameter('imgsz', 640)

        self.declare_parameter('depth_max_range', 10.0)
        self.declare_parameter('publish_rpy', True)

        self.declare_parameter('stride', 1)
        self.declare_parameter('max_points', 15000)
        self.declare_parameter('min_points', 300)

        self.declare_parameter('yaw_smoothing_alpha', 0.5)

        model_path = self.get_parameter('model_path').get_parameter_value().string_value
        self.device = self.get_parameter('device').get_parameter_value().string_value
        self.conf = float(self.get_parameter('conf').value)
        self.imgsz = int(self.get_parameter('imgsz').value)

        self.depth_max_range = float(self.get_parameter('depth_max_range').value)
        self.publish_rpy = bool(self.get_parameter('publish_rpy').value)

        self.stride = int(self.get_parameter('stride').value)
        self.max_points = int(self.get_parameter('max_points').value)
        self.min_points = int(self.get_parameter('min_points').value)
        self.alpha = float(self.get_parameter('yaw_smoothing_alpha').value)

        # Load model
        self.get_logger().info(f'Loading YOLOv8 model: {model_path}, device={self.device}')
        self.model = YOLO(model_path)
        if self.device != 'cpu':
            try:
                self.model.to(self.device)
            except Exception as e:
                self.get_logger().warn(f'Could not move model to {self.device}: {e}')

        self.bridge = CvBridge()

        self.camera_intrinsics = None
        self.latest_rgb = None
        self.latest_depth = None
        self.lock = threading.Lock()

        # prev yaw per class (store in [0,pi])
        self.prev_yaw = {0: None, 1: None, 2: None}

        # Subscribers
        self.camera_info_sub = self.create_subscription(CameraInfo, '/camera/camera/color/camera_info',self.camera_info_callback, 10)
        self.rgb_sub = self.create_subscription(Image, '/camera/camera/color/image_raw',self.rgb_callback, 10)
        self.depth_sub = self.create_subscription(Image, '/camera/camera/depth/image_rect_raw',self.depth_callback, 10)

        # Publishers
        self.pub_vis = self.create_publisher(Image, '/camera/detected_image', 10)
        self.pub_pen_position = self.create_publisher(PointStamped, '/pen_position_3d', 10)
        self.pub_box_position = self.create_publisher(PointStamped, '/box_position_3d', 10)
        self.pub_cube_position = self.create_publisher(PointStamped, '/cube_position_3d', 10)

        self.pub_pen_rpy = self.create_publisher(Float32MultiArray, '/pen_rpy', 10) if self.publish_rpy else None
        self.pub_box_rpy = self.create_publisher(Float32MultiArray, '/box_rpy', 10) if self.publish_rpy else None
        self.pub_cube_rpy = self.create_publisher(Float32MultiArray, '/cube_rpy', 10) if self.publish_rpy else None

        self.class_names = {0: 'pen', 1: 'box', 2: 'cube'}
        self.class_colors = {0: (0, 255, 0), 1: (255, 0, 0), 2: (0, 255, 255)}
        self.default_color = (0, 255, 255)

        self.detection_timer = self.create_timer(0.1, self.process_images)

    def camera_info_callback(self, msg: CameraInfo):
        if self.camera_intrinsics is None:
            self.camera_intrinsics = {'fx': msg.k[0], 'fy': msg.k[4], 'cx': msg.k[2], 'cy': msg.k[5]}
            self.get_logger().info(
                f'Camera intrinsics: fx={self.camera_intrinsics["fx"]:.1f}, '
                f'fy={self.camera_intrinsics["fy"]:.1f}, '
                f'cx={self.camera_intrinsics["cx"]:.1f}, '
                f'cy={self.camera_intrinsics["cy"]:.1f}'
            )
            self.destroy_subscription(self.camera_info_sub)
            self.camera_info_sub = None

    def rgb_callback(self, msg: Image):
        try:
            cv_image = self.bridge.imgmsg_to_cv2(msg, "bgr8")
            with self.lock:
                self.latest_rgb = cv_image
        except CvBridgeError as e:
            self.get_logger().error(f"RGB convert error: {e}")

    def depth_callback(self, msg: Image):
        """
        深度图像回调函数
        处理深度图像并转换为米为单位
        """
        try:
            depth_image = self.bridge.imgmsg_to_cv2(msg, "passthrough").astype(np.float32)
            depth_image = depth_image / 1000.0  # 转换为米
            # 检查深度图像的最大值来确认它是否在合理范围内
            depth_image[depth_image > 20.0] = 20.0  # 设置最大值，避免异常值
            # 处理无效的深度数据
            depth_image = np.nan_to_num(depth_image, nan=0.0, posinf=20.0, neginf=0.0)

            with self.lock:
                self.latest_depth = depth_image
        except CvBridgeError as e:
            self.get_logger().error(f"Depth convert error: {e}")

    #   从OBB多边形和深度图像计算3D中心点
    def _center3d_from_obb_depth(self, poly_2d: np.ndarray, depth: np.ndarray):
        # 获取图像尺寸
        H, W = depth.shape[:2]
        # ========================
        # 1. 创建OBB掩膜
        # ========================
        mask = np.zeros((H, W), dtype=np.uint8)
        cv2.fillPoly(mask, [poly_2d.astype(np.int32)], 255) # 将多边形填充为白色（255）
        # ========================
        # 2. 获取多边形内的像素坐标
        # ========================
        ys, xs = np.where(mask > 0)# 掩膜内所有像素的坐标
        if xs.size < 100:# 检查是否有足够多的像素
            return None
        # ========================
        # 3. 提取相机内参
        # ========================
        fx, fy = self.camera_intrinsics['fx'], self.camera_intrinsics['fy']
        cx, cy = self.camera_intrinsics['cx'], self.camera_intrinsics['cy']
        # ========================
        # 4. 采样深度点并转换为3D点
        # ========================
        stride = max(1, self.stride) # 采样步长，避免处理过多点
        pts = []# 存储3D点
        count = 0
        for u, v in zip(xs[::stride], ys[::stride]):  # 遍历掩膜内的像素（按步长采样）
            Z = float(depth[v, u]) # 获取深度值
            if np.isfinite(Z) and 0.0 < Z <= self.depth_max_range: # 检查深度值的有效性
                # 从像素坐标和深度计算3D坐标
                X = (float(u) - cx) * Z / fx
                Y = (float(v) - cy) * Z / fy
                pts.append([X, Y, Z])
                count += 1
                # 达到最大点数限制时停止
                if count >= self.max_points:
                    break
        # ========================
        # 5. 检查是否有足够多的有效点
        # ========================
        if len(pts) < self.min_points:
            return None
        # ========================
        # 6. 计算所有3D点的质心
        # ========================
        P = np.asarray(pts, dtype=np.float32) # N×3数组
        return np.mean(P, axis=0) # 计算均值

    def _estimate_yaw_0_pi(self, cls: int, corners: np.ndarray) -> float:
        """
        估计偏航角（0到π范围）并进行平滑处理
        """
        # 1. 从角点计算偏航角
        yaw_meas = yaw_0_to_pi_right0_left180(corners)  # [0, pi]
        # 2. 获取前一个角度
        prev = self.prev_yaw.get(cls, None)
        # 3. 确定角度周期
        # pen: 周期π（180°），因为笔有方向性
        # box/cube: 周期π/2（90°），因为正方形旋转90°后看起来相同
        if cls in (1, 2):   # box, cube
            period = (math.pi / 2.0)
        else:               # pen
            period = math.pi
        # 4. 处理第一个测量值
        if prev is None:
            yaw_out = yaw_meas
        else:

            # 5. 将当前测量值转换到等价表示范围
            # 由于yaw_meas在[0,π]，而choose_equivalent_angle期望[-π,π]
            meas_rep = yaw_meas
            if meas_rep > (math.pi / 2.0):
                meas_rep = meas_rep - math.pi  # (-pi/2, pi/2]

            prev_rep = prev
            if prev_rep > (math.pi / 2.0):
                prev_rep = prev_rep - math.pi
            # 6. 选择最接近前一个角度的等价角度
            yaw_eq = choose_equivalent_angle(meas_rep, prev_rep, period=period)
            # 7. 计算角度差并应用低通滤波
            diff = angle_diff(yaw_eq, prev_rep)
            yaw_smooth_rep = wrap_to_pi(prev_rep + self.alpha * diff)

            # 8. 转换回[0,π]范围
            yaw_out = yaw_smooth_rep
            if yaw_out < 0.0:
                yaw_out += math.pi
        # 9. 确保角度在[0,π]范围内
        yaw_out = max(0.0, min(math.pi, float(yaw_out)))
        # 10. 存储当前角度供下次使用
        self.prev_yaw[cls] = yaw_out
        return yaw_out

    def process_images(self):
        """
        主处理函数（由定时器触发，10Hz）
        执行检测、计算3D位置和姿态、发布结果
        """
        # ========================
        # 1. 检查相机内参是否就绪
        # ========================
        if self.camera_intrinsics is None:
            return
        # ========================
        # 2. 获取最新图像（线程安全）
        # ========================
        with self.lock:
            rgb = None if self.latest_rgb is None else self.latest_rgb.copy()
            depth = None if self.latest_depth is None else self.latest_depth.copy()
        # 检查图像是否有效
        if rgb is None or depth is None:
            return
        # ========================
        # 3. 运行YOLO推理
        # ========================
        try:
            # 使用YOLO模型进行预测
            # conf: 置信度阈值
            # imgsz: 输入图像尺寸（YOLO会自动resize）
            # verbose: 是否显示详细输出
            results = self.model.predict(rgb, conf=self.conf, imgsz=self.imgsz, verbose=False)
        except Exception as e:
            self.get_logger().error(f'YOLO inference error: {e}')
            return
        # ========================
        # 4. 准备可视化图像和消息头
        # ========================
        vis = rgb.copy()# 用于可视化的图像副本
        header = Header()
        header.stamp = self.get_clock().now().to_msg()# 当前时间戳
        header.frame_id = "camera_color_optical_frame"# 坐标系
        # ========================
        # 5. 初始化最佳检测结果
        # ========================
        # 每个类别只发布最高置信度的检测结果
        best_pen = None; best_pen_conf = 0.0; best_pen_rpy = None
        best_box = None; best_box_conf = 0.0; best_box_rpy = None
        best_cube = None; best_cube_conf = 0.0; best_cube_rpy = None

        r = results[0]  # 获取第一个（也是唯一一个）结果
        if not hasattr(r, 'obb') or r.obb is None or r.obb.xyxyxyxy is None:      # 检查是否有OBB检测结果
            try:
                self.pub_vis.publish(self.bridge.cv2_to_imgmsg(vis, encoding='bgr8'))   # 没有检测到物体，只发布可视化图像
            except Exception:
                pass
            return
        # 获取OBB数量
        n_obb = len(r.obb.xyxyxyxy)
        # 遍历所有检测结果
        for i in range(n_obb):
            # 6.1 提取OBB角点
            corners = try_extract_obb_corners(r, i)
            if corners is None:
                continue
            # 6.2 获取类别和置信度
            cls = int(r.obb.cls[i].item())# 类别ID
            conf = float(r.obb.conf[i].item()) # 置信度
            label = self.class_names.get(cls, f'cls{cls}') # 类别名称
            color = self.class_colors.get(cls, self.default_color) # 可视化颜色
            # 6.3 计算OBB中心（图像坐标）
            cx_pix = int(np.clip(np.mean(corners[:, 0]), 0, rgb.shape[1] - 1))
            cy_pix = int(np.clip(np.mean(corners[:, 1]), 0, rgb.shape[0] - 1))
            # 6.4 在图像上绘制OBB
            poly = corners.reshape(-1, 1, 2).astype(np.int32)
            cv2.polylines(vis, [poly], True, color, 2)
            # 绘制角点
            for p in corners:
                cv2.circle(vis, tuple(map(int, p)), 2, color, -1)
            # 绘制标签和置信度
            cv2.putText(vis, f'{label}:{conf:.2f}', (cx_pix, max(0, cy_pix - 8)),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.5, color, 1)
            # 6.5 计算3D中心点
            center3d = self._center3d_from_obb_depth(corners, depth)
            if center3d is None:
                continue
            X, Y, Z = float(center3d[0]), float(center3d[1]), float(center3d[2])
            # 6.6 计算偏航角
            yaw = self._estimate_yaw_0_pi(cls, corners)

            roll = 0.0
            pitch = 0.0
            # 6.8 在图像上显示3D位置和姿态信息
            cv2.putText(vis, f'X:{X:.2f} Y:{Y:.2f} Z:{Z:.2f} m',
                        (cx_pix, min(rgb.shape[0] - 5, cy_pix + 15)),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.45, (255, 255, 0), 1)
            cv2.putText(vis, f'Yaw:[0,180]={np.degrees(yaw):.1f} deg',
                        (cx_pix, min(rgb.shape[0] - 5, cy_pix + 30)),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.45, (0, 200, 255), 1)
            # 6.9 更新最佳检测结果
            if cls == 0 and conf > best_pen_conf:
                best_pen_conf = conf
                best_pen = (X, Y, Z)
                best_pen_rpy = (roll, pitch, yaw)

            if cls == 1 and conf > best_box_conf:
                best_box_conf = conf
                best_box = (X, Y, Z)
                best_box_rpy = (roll, pitch, yaw)

            if cls == 2 and conf > best_cube_conf:
                best_cube_conf = conf
                best_cube = (X, Y, Z)
                best_cube_rpy = (roll, pitch, yaw)
        # ========================
        # 7. 发布最佳检测结果
        # ========================
        # 7.1 发布pen的3D位置
        if best_pen is not None:
            ps = PointStamped()
            ps.header = header
            ps.point.x, ps.point.y, ps.point.z = map(float, best_pen)
            self.pub_pen_position.publish(ps)
            if self.publish_rpy and best_pen_rpy is not None:
                m = Float32MultiArray()
                m.data = list(best_pen_rpy)
                self.pub_pen_rpy.publish(m)
        # 7.2 发布box的3D位置
        if best_box is not None:
            ps = PointStamped()
            ps.header = header
            ps.point.x, ps.point.y, ps.point.z = map(float, best_box)
            self.pub_box_position.publish(ps)
            if self.publish_rpy and best_box_rpy is not None:
                m = Float32MultiArray()
                m.data = list(best_box_rpy)
                self.pub_box_rpy.publish(m)
        # 7.3 发布cube的3D位置
        if best_cube is not None:
            ps = PointStamped()
            ps.header = header
            ps.point.x, ps.point.y, ps.point.z = map(float, best_cube)
            self.pub_cube_position.publish(ps)
            if self.publish_rpy and best_cube_rpy is not None:
                m = Float32MultiArray()
                m.data = list(best_cube_rpy)
                self.pub_cube_rpy.publish(m)
        # ========================
        # 8. 发布可视化图像
        # ========================
        try:
            self.pub_vis.publish(self.bridge.cv2_to_imgmsg(vis, encoding='bgr8'))
        except Exception as e:
            self.get_logger().warn(f'publish vis failed: {e}')


def main(args=None):
    rclpy.init(args=args)
    node = YoloDetectorNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == "__main__":
    main()
```

##### 3.配置 `setup.py` 和 `package.xml`

###### 1.添加引用
```
import os
from glob import glob
```
###### 2.添加data_files
```
data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
    ],
```
###### 3.添加节点
```
    entry_points={
        'console_scripts': [
             'yolo_detector_obb = yolov8_grasping.yolo_detector_obb_node:main',
        ],
    },
```

###### 4.编辑 yolov8_grasping/package.xml，确保依赖 cv_bridge :
```
<package format="2">
  <name>yolov8_grasping</name>
  <version>0.0.0</version>
  <description>YOLOv8 ROS2 Node for Object Detection</description>
  <maintainer email="you@example.com">your_name</maintainer>
  <license>Apache-2.0</license>

  <depend>rclpy</depend>
  <depend>sensor_msgs</depend>
  <depend>cv_bridge</depend>
  <depend>depthai_ros_driver</depend>
</package>

```
#### 3.构建 ROS2 包

1.返回到 ROS2 工作空间根目录：
```
cd ~/S622_robotarm
```

2.构建工作空间：
```
colcon build --symlink-install
```

3.配置环境变量：
```
source install/setup.bash
```

#### 4.运行 ROS2 节点
```
ros2 run yolov8_grasping yolo_detector_obb --ros-args -p model_path:="/home/jasonlee/S622_robotarm/yolo-obb3.pt" -p device:="cpu"

ros2 launch depthai_ros_driver camera.launch.py 

rqt
```

#### 4.编写launch启动节点
##### 1.保存文件
```
code -p ~/S622_robotarm/src/yolov8_grasping/launch/
```
##### 2.编写yolo_detector.launch.py启动文件
```
import os
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription, TimerAction
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory


def generate_launch_description():
    # 启动参数
    model_path_arg = DeclareLaunchArgument(
        'model_path',
        # default_value='/home/jasonlee/S622_robotarm/yolo_obb.pt',
        # default_value='/home/jasonlee/S622_robotarm/yolo-obb1.pt',
        default_value='/home/jasonlee/S622_robotarm/yolo-obb3.pt',
        # default_value='/home/jasonlee/S622_robotarm/best.pt',
        description='Path to YOLOv8 model file'
    )
    
    device_arg = DeclareLaunchArgument(
        'device',
        default_value='cpu',
        description='Device for YOLOv8 inference (cpu or cuda:0)'
    )
    
    conf_threshold_arg = DeclareLaunchArgument(
        'conf',
        default_value='0.6',
        description='Confidence threshold for detections'
    )
    
    imgsz_arg = DeclareLaunchArgument(
        'imgsz',
        default_value='640',
        description='Input image size for YOLOv8'
    )

    # RealSense 相机启动
    realsense_launch = IncludeLaunchDescription(
        PythonLaunchDescriptionSource([
            os.path.join(
                get_package_share_directory("realsense2_camera"),
                "launch",
                "rs_launch.py",
            )
        ]),
        launch_arguments={
            'enable_color': 'true',
            'enable_depth': 'true',
            'depth_module.profile': '640x480x30',
            'rgb_camera.profile': '640x480x30',
            'pointcloud.enable': 'false',
            'align_depth.enable': 'true',
            'enable_sync': 'true',
            'temporal_filter.enable': 'true',
            'spatial_filter.enable': 'true',
            'hole_filling_filter.enable': 'true',
        }.items()
    )

    # YOLOv8 检测节点（延迟启动）
    yolo_detector_node = TimerAction(
        period=3.0,
        actions=[
            Node(
                package='yolov8_grasping',
                executable='yolo_detector',
                name='yolov8_detector',
                output='screen',
                parameters=[{
                    'model_path': LaunchConfiguration('model_path'),
                    'device': LaunchConfiguration('device'),
                    'conf': LaunchConfiguration('conf'),
                    'imgsz': LaunchConfiguration('imgsz'),
                }]
            )
        ]
    )
    yolo_detector_node1 = TimerAction(
        period=3.0,
        actions=[
            Node(
                package='yolov8_grasping',
                executable='yolo_detector1',
                name='yolov8_detector1',
                output='screen',
                parameters=[{
                    'model_path': LaunchConfiguration('model_path'),
                    'device': LaunchConfiguration('device'),
                    'conf': LaunchConfiguration('conf'),
                    'imgsz': LaunchConfiguration('imgsz'),
                }]
            )
        ]
    )
    yolo_detector_node_obb = TimerAction(
        period=3.0,
        actions=[
            Node(
                package='yolov8_grasping',
                executable='yolo_detector_obb',
                name='yolov8_detector_obb',
                output='screen',
                parameters=[{
                    'model_path': LaunchConfiguration('model_path'),
                    'device': LaunchConfiguration('device'),
                    'conf': LaunchConfiguration('conf'),
                    'imgsz': LaunchConfiguration('imgsz'),
                }]
            )
        ]
    )


    return LaunchDescription([
        model_path_arg,
        device_arg,
        conf_threshold_arg,
        imgsz_arg,
        realsense_launch,
        # yolo_detector_node,
        # yolo_detector_node1,
        yolo_detector_node_obb,
    ])



```
##### 3.启动launch
```
source install/setup.bash
ros2 launch yolov8_grasping yolo_detector.launch.py 
```
##### 4.关系说明
```
终端命令: ros2 run yolov8_grasping yolo_detector
           ↓
启动可执行文件: yolo_detector (setup.py中定义)
    entry_points={
        'console_scripts': [
            'yolo_detector = yolov8_grasping.yolo_detector_node:main',
            'yolo_detector1 = yolov8_grasping.yolo_detector_node1:main',
            'yolo_detector_obb = yolov8_grasping.yolo_detector_obb_node:main',
        ],
    },

           ↓  
执行Python文件: yolov8_grasping/yolo_detector_node.py
           ↓
创建ROS2节点: 'yolov8_detector' (代码中super().__init__()定义)

```
