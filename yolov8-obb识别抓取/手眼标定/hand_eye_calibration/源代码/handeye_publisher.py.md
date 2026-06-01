```
#!/usr/bin/env python3
import numpy as np
import rclpy
import tf2_ros
from easy_handeye2.handeye_calibration import load_calibration
from geometry_msgs.msg import Transform, TransformStamped
from rclpy.node import Node
from rclpy.time import Time
from rclpy.executors import ExternalShutdownException
from rclpy.node import ParameterType, ParameterDescriptor
from transforms3d.quaternions import quat2mat, mat2quat
- **功能**：导入必要的Python库和ROS2模块
- **关键组件**：
    - `numpy`：数值计算
    - `rclpy`：ROS2 Python客户端库
    - `tf2_ros`：ROS坐标变换处理
    - `load_calibration`：从文件加载手眼标定数据
    - `Transform/TransformStamped`：ROS几何变换消息
    - `transforms3d`：四元数与矩阵转换工具
```

```
def transform_to_matrix(transform):
    # 将ROS Transform消息转换为4x4齐次变换矩阵
    translation = np.array([...])  # 提取平移向量
    rotation = np.array([...])     # 提取四元数
    rotation_matrix = quat2mat(rotation)  # 四元数→旋转矩阵
    matrix = np.eye(4)             # 创建4x4单位矩阵
    matrix[:3, :3] = rotation_matrix  # 填充旋转部分
    matrix[:3, 3] = translation     # 填充平移部分
    return matrix

def matrix_to_transform(matrix):
    # 将4x4齐次变换矩阵转换为ROS Transform消息
    rotation_matrix = matrix[:3, :3]  # 提取旋转矩阵
    translation = matrix[:3, 3]       # 提取平移向量
    rotation = mat2quat(rotation_matrix)  # 旋转矩阵→四元数
    transform = Transform()           # 创建Transform对象
    # 填充平移和旋转数据
    return transform

坐标变换辅助函数
- **功能**：实现ROS Transform消息与NumPy矩阵的相互转换
- **数学基础**：使用齐次坐标表示3D空间变换
```

```
class HandeyePublisher(Node):
    def __init__(self):
        super().__init__('handeye_publisher')
        
        # 声明并获取标定名称参数
        calibration_name_p = self.declare_parameter('calibration_name', '')
        name = calibration_name_p.value
        
        # 加载手眼标定数据
        self.calibration = load_calibration(name)
        parameters = self.calibration.parameters
        
        # 获取标记ID参数
        self.marker_id = self.declare_parameter("marker_id", 1).value
        
        # 确定坐标框架类型
        if parameters.calibration_type == 'eye_in_hand':
            orig = parameters.robot_effector_frame  # 眼在手上：机械臂末端
        else:
            orig = parameters.robot_base_frame      # 眼在手外：机械臂基座
        dest = "camera_link"  # 目标坐标系
        
        # TF2相关初始化
        self.tf_buffer = tf2_ros.Buffer()
        self.tf_listener = tf2_ros.TransformListener(self.tf_buffer, self)
        self.broadcaster = tf2_ros.StaticTransformBroadcaster(self)
        
        # 初始化变换消息
        self.calibration_tf = TransformStamped()
        self.calibration_tf.header.frame_id = orig
        self.calibration_tf.child_frame_id = dest
        
        # 创建定时器
        self.compute_transform_timer = self.create_timer(0.1, self.compute_transform)

主节点类 HandeyePublisher
```

```
    def compute_transform(self):
        try:
            # 获取相机到标定板的变换
            color_to_link_tf = self.tf_buffer.lookup_transform(
                target_frame=self.calibration.parameters.tracking_base_frame,
                source_frame="camera_link",
                time=Time())
        
        # 转换标定数据和实时TF为矩阵
        base_to_color_mat = transform_to_matrix(self.calibration.transform)
        color_to_link_mat = transform_to_matrix(color_to_link_tf.transform)
        
        # 计算组合变换：机械臂基座→相机
        base_to_link_mat = np.dot(base_to_color_mat, color_to_link_mat)
        self.calibration_tf.transform = matrix_to_transform(base_to_link_mat)
        
        # 切换为发布模式
        self.compute_transform_timer.cancel()
        self.publish_transform_timer = self.create_timer(0.1, self.publish_transform)

变换计算回调函数
```

```
    def publish_transform(self):
        # 更新时戳并发布静态变换
        self.calibration_tf.header.stamp = self.get_clock().now().to_msg()
        self.broadcaster.sendTransform(self.calibration_tf)

变换发布回调函数
```

```
def main(args=None):
    rclpy.init(args=args)
    handeye_publisher = HandeyePublisher()
    try:
        rclpy.spin(handeye_publisher)
    finally:
        handeye_publisher.destroy_node()

if __name__ == '__main__':
    main()

主函数
```

 可视化逻辑图表
![[Pasted image 20250725004915.png]]

功能模块化结构
![[deepseek_mermaid_20250724_ffef5c.svg]]

### 核心功能实现逻辑

#### 1. **初始化阶段**：
    - 加载预存的手眼标定数据（`eye_in_hand`或`eye_to_hand`）
    - 根据标定类型确定源坐标系（机械臂末端/基座）
    - 初始化TF2系统（缓冲器/监听器/广播器）
#### 2. **变换计算阶段**：
![[deepseek_mermaid_20250724_143901.svg]]
- 组合标定数据与实时相机-标定板变换
- 得到机械臂坐标系→相机坐标系的稳定变换

#### 3 **持续发布阶段**
    - 以10Hz频率广播静态TF变换
    - 使机械臂坐标系与相机坐标系在TF树中关联
    - 下游节点可直接查询两坐标系间关系

### 功能总结

1. **核心功能**：将离线标定结果与实时传感器数据融合，建立机械臂与相机间的精确坐标变换关系
2. **关键创新点**：
    - 支持两种标定类型（眼在手上/眼在手外）
    - 自动切换计算/发布模式
    - 使用静态TF提高坐标变换效率