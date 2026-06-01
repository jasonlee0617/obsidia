## 一、系统概述

### 1.1 项目目标

本项目实现了一个基于ROS2的机械臂视觉抓取系统，通过相机检测目标物体（笔、立方体、盒子），计算其3D位置和姿态，然后控制机械臂完成抓取和放置任务。

### 1.2 技术栈

- **ROS2 Humble**：机器人操作系统
- **YOLOv8-OBB**：有向边界框目标检测
- **MoveIt2**：运动规划与控制
- **RealSense D435**：深度相机
- **TF2**：坐标变换系统
## 二、系统架构

### 2.1 节点结构

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  YOLO检测节点    │    │  抓取控制节点    │    │  MoveIt2服务器   │
│ (Python)        │────▶ (Python)        │────▶ (C++)          │
│ - 目标检测       │    │ - 状态机         │    │ - 运动规划      │
│ - 姿态估计       │    │ - 路径规划       │    │ - 碰撞检测      │
│ - 3D坐标计算     │    │ - 坐标变换       │    │ - 轨迹执行      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                         │
         ▼                         ▼
┌─────────────────┐    ┌─────────────────┐
│  相机驱动        │    │  机械臂控制器   │
│ (RealSense)     │    │ (ros2_control)  │
│ - RGB图像        │    │ - 关节控制      │
│ - 深度图像       │    │ - 夹爪控制      │
└─────────────────┘    └─────────────────┘
```

### 2.2 通信话题

```
检测节点 → 抓取节点：
  /pen_position_3d    (PointStamped)    # 笔的3D位置
  /cube_position_3d  (PointStamped)    # 立方体的3D位置  
  /box_position_3d   (PointStamped)    # 盒子的3D位置
  /pen_rpy           (Float32MultiArray) # 笔的欧拉角
  /cube_rpy          (Float32MultiArray) # 立方体的欧拉角
  /detected_image    (Image)            # 可视化图像

抓取节点 → MoveIt2：
  /planning_scene    (PlanningScene)    # 规划场景更新
  /collision_object  (CollisionObject)  # 碰撞物体
  /trajectory_execution_event (String)  # 轨迹执行事件

用户接口：
  /manual_abort      (Bool)            # 手动中断信号
  /task_state        (String)          # 任务状态发布
```

## 三、核心原理详解

### 3.1 相机坐标系到机械臂坐标系的变换

#### 3.1.1 相机内参与3D重建

```
# 从像素坐标(u,v)和深度Z计算3D坐标(X,Y,Z)
# 相机内参：fx, fy（焦距），cx, cy（光心）
X = (u - cx) * Z / fx
Y = (v - cy) * Z / fy
Z = depth_value
```

#### 3.1.2 手眼标定与TF变换

```
def camera_to_base_transform(self, camera_point_stamped):
    # 1. 创建PoseStamped消息（位置来自检测，姿态设为单位四元数）
    camera_pose = PoseStamped()
    camera_pose.header = camera_point_stamped.header
    camera_pose.pose.position = camera_point_stamped.point
    camera_pose.pose.orientation.w = 1.0
    
    # 2. 查询TF变换（从相机坐标系到基座坐标系）将相机坐标系下的点变换到基座坐标系
    tf = self.tf_buffer.lookup_transform(
        self.base_frame,           # 目标坐标系：base_link
        camera_point_stamped.header.frame_id,  # 源坐标系：camera_color_optical_frame
        rclpy.time.Time.from_msg(camera_point_stamped.header.stamp)
    )
    
    # 3. 应用坐标变换
    base_pose = tf2_geometry_msgs.do_transform_pose_stamped(camera_pose, tf)
    return base_pose.pose.position
```

- **手眼标定**：确定相机与机械臂基座的相对位置关系
- **TF树**：ROS中维护的坐标变换关系链
- **时间同步**：使用消息的时间戳查询对应时刻的变换

```
          世界/基座坐标系 (base_link)
               ↑
               | 变换 T_base_camera
               ↓
          相机坐标系 (camera_color_optical_frame)
               ↑
               | 物体在相机坐标系中的位置 P_camera
               ↓
             物体 (笔/立方体)
```
### 3.2 目标姿态计算与处理

#### 3.2.1 OBB（有向边界框）角度计算

```
def yaw_0_to_pi_right0_left180(corners_2d):
    # 1. 找出OBB的最长边（代表物体的"右侧"）
    # 2. 计算该边与x轴的夹角
    # 3. 将角度限制在[0, π]范围内
    
    # 物理意义：
    # 0°: 物体右侧朝右
    # 90°: 物体右侧朝下  
    # 180°: 物体右侧朝左
```

#### 3.2.2 角度周期性与等价角度选择

```
def choose_equivalent_angle(cur, prev, period):
    # 处理角度歧义问题：
    # pen: 周期π（180°），因为笔旋转180°后看起来一样
    # box/cube: 周期π/2（90°），因为立方体旋转90°后看起来一样
    
    # 算法：生成多个等价角度候选，选择最接近前一个角度的
    candidates = []
    for k in (-4, -3, -2, -1, 0, 1, 2, 3, 4):
        cand = cur + k * period
        candidates.append(cand)
    
    # 选择与prev差异最小的角度
    return min(candidates, key=lambda x: abs(angle_diff(x, prev)))
```

### 3.3 状态机设计

#### 3.3.1 状态定义

```
class TaskState(Enum):
    IDLE = "idle"                     # 空闲状态，准备开始
    SEARCHING = "searching"           # 搜索目标物体
    MOVING_TO_TARGET_ABOVE = "moving_to_target_above"  # 移动到目标上方
    MOVING_TO_TARGET = "moving_to_target"              # 下降到抓取位置
    GRASPING = "grasping"             # 执行抓取动作
    LIFTING_TARGET = "lifting_target" # 抬起物体
    MOVING_TO_BOX = "moving_to_box"   # 移动到盒子位置
    RELEASING = "releasing"           # 释放物体
    RETURNING_HOME = "returning_home" # 返回初始位置
    COMPLETED = "completed"           # 任务完成
    ERROR = "error"                   # 错误状态
```

#### 3.3.2 状态转移逻辑

```
IDLE → SEARCHING → MOVING_TO_TARGET_ABOVE → MOVING_TO_TARGET
      ↓                                      ↑
  (等待目标)                              (失败)
  
MOVING_TO_TARGET → GRASPING → LIFTING_TARGET → MOVING_TO_BOX
      ↓                                         ↑
  (失败)                                     (失败)
  
MOVING_TO_BOX → RELEASING → RETURNING_HOME → COMPLETED → IDLE
      ↓                       ↑
  (失败)                 (失败/超时)
```

### 3.4 路径规划与代价函数

#### 3.4.1 多路径规划与选择

```
def select_best_path(self, paths):
    best = None
    best_cost = float("inf")
    
    for traj in paths:
        # 计算总代价 = 关节路径长度 + 权重 × 腕部关节路径长度
        cost = (self.calculate_path_length(traj) + 
                self.WRIST_WEIGHT * self.calculate_wrist_path_length(traj))
        
        if cost < best_cost:
            best_cost = cost
            best = traj
    
    return best
```

**路径代价的物理意义**：
- **关节路径长度**：所有关节移动的总距离，越小越节能
- **腕部关节路径长度**：腕部关节（j3, j4, j5）的移动距离，过大可能导致奇异点
- **权重平衡**：通过`WRIST_WEIGHT`参数平衡两个代价的优先级

#### 3.4.2 关节约束设置

```
# 设置j2关节约束，避免奇异位形
self.moveit2_arm.set_path_joint_constraint(
    joint_positions=[-1.5708],  # -90°位置
    joint_names=["j2"],        # 第二个关节
    tolerance=1.5708,          # ±90°范围
    weight=1.0                # 约束强度
)
```

### 3.5 碰撞避免机制

#### 3.5.1 Keepout区域

```
def enable_keepout(self, z_min: float):
    # 在桌面高度创建一个碰撞体，防止机械臂撞到桌面
    # 参数：
    # - z_min: 桌面高度（机械臂不能低于这个高度）
    # - KEEP_OUT_THICKNESS: 碰撞体厚度
    # - KEEP_OUT_XY_SIZE: 碰撞体平面尺寸
    
    # 在以下状态启用：
    # 1. MOVING_TO_TARGET_ABOVE: 从高处接近目标
    # 2. MOVING_TO_BOX: 从高处接近盒子
    
    # 在以下状态禁用：
    # 1. MOVING_TO_TARGET: 需要下降到抓取高度
```

#### 3.5.2 碰撞体管理流程

```
启用Keepout → 发布CollisionObject → MoveIt2更新规划场景
                    ↓
             机械臂规划时会避开该区域
                    ↓
禁用Keepout → 移除CollisionObject → 允许机械臂进入该区域
```

### 3.6 运动控制参数

#### 3.6.1 速度与加速度配置

```
# 不同运动阶段使用不同的速度参数
运动阶段                   最大速度    最大加速度
---------------------------------------------------
高处移动 (MOVING_TO_TARGET_ABOVE)   0.06     0.06
下降抓取 (MOVING_TO_TARGET)        0.01     0.01
抬升物体 (LIFTING_TARGET)          0.05     0.05
移动到盒子 (MOVING_TO_BOX)         0.10     0.10
返回Home (RETURNING_HOME)          0.15     0.15
```

#### 3.6.2 抓取姿态配置

```
self.grasp_profile = {
    TargetType.PEN: {
        "roll": 0.0,          # 绕x轴旋转
        "pitch": -180.0,      # 绕y轴旋转（-180°使夹爪朝下）
        "yaw_offset": -125.0, # 绕z轴偏移（适应笔的方向）
        "above_z": 0.03,      # 安全高度
        "grasp_z": 0.00,      # 抓取高度
    },
    TargetType.CUBE: {
        "roll": 0.0,
        "pitch": -180.0,
        "yaw_offset": -135.0,
        "above_z": 0.05,
        "grasp_z": 0.01,
    },
}
```

### 3.7 中断与恢复机制

#### 3.7.1 中断检测与处理

```
def _wait_moveit_idle_or_abort(self, m, action_name, timeout_sec):
    # 轮询等待运动完成，同时检查中断信号
    while rclpy.ok():
        # 1. 检查中断事件
        if self.abort_event.is_set():
            self._cancel_all_motion_now()  # 立即停止所有运动
            return False
        
        # 2. 检查运动状态
        state = m.query_state()
        if state == MoveIt2State.IDLE:
            return True
        
        # 3. 检查超时
        if time.time() - start_time > timeout_sec:
            self._cancel_all_motion_now()
            return False
```

#### 3.7.2 安全恢复流程

```
中断触发 → 停止所有运动 → 打开夹爪 → 禁用Keepout → 
返回Home位置 → 清空状态 → 重新开始搜索
```

## 四、算法优化与调试技巧

### 4.1 检测稳定性优化

#### 4.1.1 深度数据滤波

```
# 使用中值滤波和标准差过滤深度异常值
depth_median = np.median(depths_valid)
depth_std = np.std(depths_valid)

# 只保留在[median-std, median+std]范围内的点
valid_mask = (depths_valid > depth_median - depth_std) & \
             (depths_valid < depth_median + depth_std)
```

#### 4.1.2 角度平滑滤波

```
# 应用指数平滑滤波
def smooth_angle(current, previous, alpha):
    # alpha: 平滑系数（0~1），越小越平滑
    diff = angle_diff(current, previous)
    return wrap_to_pi(previous + alpha * diff)
```

### 4.2 运动规划优化

#### 4.2.1 多候选规划

```
# 生成多个规划候选，选择代价最小的
paths = []
for _ in range(self.NUM_CANDIDATE_PLANS):  # 默认5个
    plan = self.moveit2_arm.plan(target_pose, cartesian=False)
    if plan:
        paths.append(plan)

if paths:
    best_path = self.select_best_path(paths)
```

#### 4.2.2 轨迹重定时

```
# 调整轨迹的速度分布，确保平滑执行
retimed_trajectory = self.moveit2_arm._retime_trajectory_if_needed(
    trajectory, 
    cartesian=cartesian
)
```

## 5.遗留问题
### 5.1 检测不稳定问题

- **现象**：目标位置和姿态跳动
- **解决方案**：
    1. 增加检测置信度阈值
    2. 应用卡尔曼滤波器或多帧平均
    3. 检查相机曝光和光照条件

### 5.2 抓取精度不足

- **现象**：抓取位置偏移
- **解决方案**：
    1. 重新标定手眼矩阵
    2. 优化深度数据质量（增加结构光）
    3. 使用更高分辨率的相机