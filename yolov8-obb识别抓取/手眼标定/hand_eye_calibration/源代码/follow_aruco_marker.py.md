```
#!/usr/bin/env python3
"""
A script to follow an aruco marker with a robot arm using PyMoveit2.
"""
import rclpy
from rclpy.callback_groups import ReentrantCallbackGroup
from rclpy.executors import MultiThreadedExecutor
from rclpy.node import Node
from rclpy.time import Time
import tf2_ros
from geometry_msgs.msg import PoseStamped
from geometry_msgs.msg import Pose
from ros2_aruco_interfaces.msg import ArucoMarkers
from tf2_geometry_msgs import do_transform_pose
from pymoveit2 import MoveIt2
- **导入必要的ROS 2库**：
    - `rclpy`：ROS 2 Python客户端库
    - `ReentrantCallbackGroup`：允许回调函数重入的回调组
    - `MultiThreadedExecutor`：多线程执行器
    - `Node`：ROS 2节点基类
    - `Time`：ROS时间处理
- **导入TF2相关库**：
    - `tf2_ros`：TF2库，处理坐标变换
    - `PoseStamped`和`Pose`：几何位姿消息类型
    - `do_transform_pose`：执行坐标变换的函数
- **导入应用相关库**：
    - `ArucoMarkers`：ArUco标记检测消息类型
    - `MoveIt2`：MoveIt 2机械臂控制接口
```

```
class ArucoMarkerFollower(Node):

    def __init__(self):
        super().__init__("aruco_marker_follower")
        self.logger = self.get_logger()
  
    - **定义节点类**：
    - 继承自`Node`
    - 节点名称："aruco_marker_follower"
    - 获取日志记录器
```

```
        self.arm_joint_names = [
            "joint1", "joint2", "joint3", "joint4", "joint5", "joint6"
        ]
        self.moveit2 = MoveIt2(
            node=self,
            joint_names=self.arm_joint_names,
            base_link_name="base_link",
            end_effector_name="link6_1",
            group_name="dummy2_arm",
            callback_group=ReentrantCallbackGroup(),
        )
        self.moveit2.planner_id = "RRTConnect"
        self.moveit2.max_velocity = 1.0
        self.moveit2.max_acceleration = 1.0
- **机械臂配置**：
    - 定义关节名称列表
    - 初始化MoveIt2接口
    - 设置规划器参数（RRTConnect算法）
    - 设置最大速度和加速度
```

RRTConnect规划器工作流程
![[deepseek_mermaid_20250726_9815c5.svg]]
在系统中的角色：
- **输入**：起始位姿（当前关节状态）+ 目标位姿（标记上方5cm）
- **输出**：时间参数化的关节轨迹

```
        # ID of the aruco marker mounted on the robot
        self.marker_id = self.declare_parameter(
            "marker_id", 1).get_parameter_value().integer_value

        self.subscription = self.create_subscription(ArucoMarkers,
                                                     "/aruco_markers",
                                                     self.handle_aruco_markers,
                                                     1)

- **参数和订阅器**：
    - 声明并获取ArUco标记ID参数（默认值1）
    - 创建订阅器，监听"/aruco_markers"话题
```

```
        self.pose_pub = self.create_publisher(PoseStamped, "/cal_marker_pose",
                                              1)

        self.target_pose_pub = self.create_publisher(
            PoseStamped, "/follow_aruco_target_pose", 1)

- **发布器配置**：
    - 发布校正后的标记位姿
    - 发布目标跟随位姿
```

```
        self.tf_buffer = tf2_ros.Buffer()
        self.tf_listener = tf2_ros.TransformListener(self.tf_buffer, self)
        self._prev_marker_pose = None

- **TF2配置**：
    - 创建TF2缓冲区和监听器
    - 初始化上一次标记位姿变量
```

```
    def handle_aruco_markers(self, msg: ArucoMarkers):
        cal_marker_pose = None
        for i, marker_id in enumerate(msg.marker_ids):
            if marker_id == self.marker_id:
                cal_marker_pose = msg.poses[i]
                break
            else:
                self.logger.info(f"Detected unexpected marker with ID: {marker_id}")

- **ArUco标记处理回调**：
    - 遍历检测到的所有标记
    - 寻找与目标ID匹配的标记
    - 记录匹配标记的位姿
```

```
        if cal_marker_pose is None:
            return

        # only start following if the marker pose has changed by at least 2cm
        if self._prev_marker_pose is not None:
            if ((cal_marker_pose.position.x -
                 self._prev_marker_pose.position.x)**2 +
                (cal_marker_pose.position.y -
                 self._prev_marker_pose.position.y)**2 +
                (cal_marker_pose.position.z -
                 self._prev_marker_pose.position.z)**2 > 0.02**2):
                self._prev_marker_pose = cal_marker_pose
                return

        self._prev_marker_pose = cal_marker_pose

- **位姿变化检测**：
    - 检查标记位姿是否变化超过2cm
    - 更新上一次位姿记录
    - 如果变化超过阈值则跳过本次跟随（避免频繁移动）
```

```
        # get pose in robot base frame
        try:
            transformed_pose = self._transform_pose(cal_marker_pose,
                                                    "camera_color_optical_frame",
                                                    "base_link")
        except tf2_ros.LookupException as e:
            self.logger.error(f"Error transforming pose: {e}")
            return

        self.logger.info(f"+++++++++++Following marker at pose: {transformed_pose}")
        self.move_to(transformed_pose)

- **坐标变换和移动**：
    - 将标记位姿从相机坐标系变换到机器人基坐标系
    - 记录变换后的位姿
    - 调用移动函数
```

```
    def _transform_pose(self, pose: Pose, source_frame,
                        target_frame: str) -> Pose:
        # Get the transform from source frame to target frame
        transform = self.tf_buffer.lookup_transform(target_frame, source_frame,
                                                    Time())
        # Transform the pose
        transformed_pose = do_transform_pose(pose, transform)
        # publish pose
        stamped_pose = PoseStamped()
        stamped_pose.header.frame_id = target_frame
        stamped_pose.pose = transformed_pose
        self.pose_pub.publish(stamped_pose)

        pose.position.z += 0.05
        transformed_pose = do_transform_pose(pose, transform)

        stamped_pose = PoseStamped()
        stamped_pose.header.frame_id = target_frame
        stamped_pose.pose = transformed_pose
        self.target_pose_pub.publish(stamped_pose)
        return transformed_pose

- **位姿变换函数**：
    - 获取坐标系间变换
    - 应用变换到标记位姿
    - 发布校正后的位姿
    - 在Z轴增加5cm偏移（避免碰撞）
    - 发布目标跟随位姿
```

```
    def move_to(self, msg: Pose):
        pose_goal = PoseStamped()
        pose_goal.header.frame_id = "base_link"
        pose_goal.pose = msg

        self.moveit2.move_to_pose(pose=pose_goal)
        self.moveit2.wait_until_executed()

- **机械臂移动函数**：
    - 创建目标位姿消息
    - 调用MoveIt2执行移动
    - 等待移动完成
```

```
def main():
    rclpy.init()
    node = ArucoMarkerFollower()
    executor = MultiThreadedExecutor(4)
    executor.add_node(node)
    try:
        executor.spin()
    except KeyboardInterrupt:
        pass
    rclpy.shutdown()

if __name__ == "__main__":
    main()

- **主函数**：
    - 初始化ROS 2
    - 创建节点实例
    - 使用多线程执行器（4个线程）
    - 运行节点直到中断
    - 关闭ROS 2
```

![[Pasted image 20250725003019.png]]


![[deepseek_mermaid_20250724_749e2b.svg]]

### 功能实现分析

#### 1. 核心功能
- **视觉-机械臂闭环控制**：实现基于视觉标记的机械臂实时跟随
- **坐标系统一**：将相机检测的标记位姿转换到机器人基坐标系
- **智能跟随策略**：
    - 仅当标记移动超过2cm时才触发跟随
    - 在标记上方5cm处跟随（避免碰撞）
    - 使用RRTConnect算法进行运动规划
        

#### 2. 模块功能

| 模块         | 功能        | 关键组件                       |
| ---------- | --------- | -------------------------- |
| **感知模块**   | 检测ArUco标记 | ros2_aruco_interfaces      |
| **坐标变换模块** | 坐标系间转换    | tf2_ros, do_transform_pose |
| **运动规划模块** | 路径规划与控制   | MoveIt2, RRTConnect        |
| **决策模块**   | 跟随策略实现    | 位姿变化检测, Z轴偏移               |
| **通信模块**   | ROS话题管理   | rclpy, 发布/订阅               |

#### 3. 功能实现逻辑
##### 1. **系统初始化**：
    - 加载机械臂参数
    - 配置MoveIt2接口
    - 建立TF2监听
##### 2. **标记检测循环**：
    - 持续监听ArUco检测结果
    - 筛选目标标记位姿
    - 评估位姿变化幅度
##### 3. **坐标变换处理**：
    transform = tf_buffer.lookup_transform("base_link", "camera_frame", Time())
    transformed_pose = do_transform_pose(original_pose, transform)
    - 获取相机到基座的变换
    - 应用变换到标记位姿
##### 4. **跟随策略执行**：
    - 添加安全高度偏移（Z+0.05m）
    - 发布目标位姿用于可视化
    - 调用MoveIt2执行运动规划
##### 5. **运动控制**：
    - 使用RRTConnect算法规划路径
    - 以1.0速度/加速度执行移动
    - 等待动作完成

#### 4. 创新设计特点
- **变化阈值过滤**：减少不必要的机械臂运动
- **安全高度偏移**：防止机械臂碰撞标记
- **多线程处理**：使用ReentrantCallbackGroup处理并发
- **可视化支持**：发布中间位姿用于调试
- **参数化配置**：可动态调整标记ID