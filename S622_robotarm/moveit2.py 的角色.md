### 1. 这份 moveit2.py 的角色：MoveIt2 通信封装层
- 对外提供：`move_to_pose()`、`move_to_configuration()`、`plan()`、`execute()`、`compute_fk()`、`compute_ik()`、各种约束/碰撞体 API
- 对内通过 ROS2：
    - **订阅** `joint_states` 获取当前状态
    - **服务**：
        - `plan_kinematic_path`（GetMotionPlan）——关节空间规划（OMPL 等）
        - `compute_cartesian_path`（GetCartesianPath）——笛卡尔插值规划
        - `compute_fk`（GetPositionFK）——正解
        - `compute_ik`（GetPositionIK）——逆解
        - `get_planning_scene` / `apply_planning_scene`——场景管理
    - **Action**：
        - `move_action`（MoveGroup）——（可选）规划+执行一体化
        - `execute_trajectory`（ExecuteTrajectory）——执行 MoveIt 轨迹
- 并且提供 collision object 的话题接口：
    - `/collision_object`
    - `/attached_collision_object`

 核心认知：**这个 Python 文件不自己做 IK/FK/规划/碰撞检测**，它只是把请求消息发给MoveIt2 的服务与 action。
 
### 2. `move_to_pose()`：每次抓取调用的入口
#### 2.1 把输入统一成 PoseStamped
它允许你传：
- `PoseStamped`
- `Pose`
- 或 `position + quat_xyzw`
然后它会统一成：
```
pose_stamped = PoseStamped(header=Header(stamp=now, frame_id=base_link), pose=Pose(...))
```
这一步很关键：**MoveIt 的所有规划/IK 都要知道目标在哪个坐标系下**（frame_id）
#### 2.2 关键分支：`__use_move_group_action` + `cartesian` 决定走哪条“管线”
##### 分支 A：`use_move_group_action=True` 且 `cartesian=False`
**直接给 move_group 一个 MoveGroup action goal，让它内部规划并执行**。
执行流程：
1. `set_pose_goal(...)`  
    这会把**位置约束** + **姿态约束**塞进 `self.__move_action_goal.request.goal_constraints[-1]` 中
2. 设置 `start_state` 为当前 `joint_states`
3. `_send_goal_async_move_action()` 把 goal 发给 `/move_action` action server
4. 等 move_group 返回结果（成功/失败）

可以把它理解成：

Python：我给你“目标 + 起点 + 参数”，你帮我搞定规划与执行。

**优点**：简单，move_group 负责全流程（包括碰撞检查、时间参数化、执行监控）。  
**缺点**：你拿不到中间的轨迹（除非用 feedback/结果里的内容或另走 service）。
##### 分支 B：其他情况（包括 cartesian=True，或 use_move_group_action=False）
这一条是“两段式”：
1. `self.plan(...)`：通过 service 规划并拿到 `JointTrajectory`
2. `self.execute(trajectory)`：通过 `execute_trajectory` action 把轨迹交给 MoveIt 执行器
    

可以把它理解成：
 
Python：我先去“问 move_group 要一条轨迹”，再“把轨迹交给执行器去跑”。

这个分支：
- 自己检查/修改轨迹
- 把轨迹保存/复用  
### 3. 目标是如何变成“约束”的：`set_position_goal` / `set_orientation_goal`

==这些函数就是“把你的期望写进 MotionPlanRequest”：==
#### 3.1 PositionConstraint（球形容差）

`create_position_constraint()`：
- `constraint.header.frame_id = base_link（默认）`
- `constraint.link_name = end_effector（默认）`
- 目标位置写入 `primitive_poses[0].position`
- 容差被做成一个 Sphere primitive：半径 = `tolerance`
- 
==等价于：“末端连杆的参考点必须落在以目标点为中心、半径 tolerance 的球里”==

#### 3.2 OrientationConstraint（XYZ 轴容差）

`create_orientation_constraint()`：
- `constraint.orientation = quat_xyzw`
- `absolute_x/y/z_axis_tolerance` = 容差
- `parameterization` 决定用欧拉误差还是旋转向量
    
==等价于：“末端姿态要接近目标姿态，允许一定角度误差”==

最终它们都被 append 到：
```
self.__move_action_goal.request.goal_constraints[-1].position_constraints
self.__move_action_goal.request.goal_constraints[-1].orientation_constraints
```

也就是：**MotionPlanRequest.goal_constraints**

### 4. 规划是怎么发生的：`plan_async()` -> `_plan_kinematic_path()` / `_plan_cartesian_path()`

#### 4.1 `plan_async()` 做了什么
1. 根据输入设置目标：
    - pose -> set_position_goal + set_orientation_goal
    - joint_positions -> set_joint_goal
2. 设置起点：
    - 如果你传 `start_joint_state` 就用它
    - 否则用 `joint_states` 订阅来的当前状态
3. 调规划服务：
    - cartesian=True：`compute_cartesian_path`
    - 否则：`plan_kinematic_path`
4. 规划请求发出去后，会 `clear_goal_constraints()` 和 `clear_path_constraints()`，避免下一次调用被污染

这一步很关键：**这个类是“可复用的”，所以它每次规划后清空约束，保证下一次从干净状态开始**

#### 4.2 `_plan_kinematic_path()`：调用 `GetMotionPlan` 服务

这里做的是,它复用你同一个 `MoveGroup.Goal().request`，把它塞进 `GetMotionPlan.Request`
```
self.__kinematic_path_request.motion_plan_request = self.__move_action_goal.request
```

然后给 header stamp 填上当前时间，最后：
```
return self._plan_kinematic_path_service.call_async(self.__kinematic_path_request)
```

**真正规划发生在哪里？**
- 在 MoveIt2 的 move_group 节点内部（planning pipeline：OMPL/CHOMP 等 + 碰撞检测 + 约束处理）,Python 只是发请求、收结果。
#### 4.3 `_plan_cartesian_path()`：调用 `GetCartesianPath` 服务

Cartesian 路径不是 OMPL，而是 MoveIt 的“笛卡尔插值 + IK + 碰撞检查”流程。
这个函数构造 `GetCartesianPath.Request`：
- 起点 = start_state
- group_name、link_name
- max_step（每步插值距离）
- path_constraints 也可以带进去
- waypoints：这里仅用一个目标点（所以是“从当前到目标”的直线插值）
    
它返回的结果里有：
- `fraction`：笛卡尔路径完成比例（0~1）
- `solution.joint_trajectory`

这就是 `get_trajectory()` 对 cartesian 的那段判断：fraction 不够就拒绝

### 5. 执行是怎么发生的：`execute()` -> ExecuteTrajectory action

**`execute()` 不直接发给 ros2_control 的 FollowJointTrajectory，而是发给：`ExecuteTrajectory` action（action name: `"execute_trajectory"`）**

构造的 goal ,然后 `_send_goal_async_execute_trajectory()` 发给 action server
```
ExecuteTrajectory.Goal()
goal.trajectory.joint_trajectory = joint_trajectory
```

**谁是 execute_trajectory 的 server？**
- 通常是 MoveIt 的 trajectory execution manager（在 move_group/或 moveit_ros_control_interface 相关组件里）
- 它会把轨迹拆成控制器可执行的 JointTrajectory，并发送到 ros2_control 的 joint_trajectory_controller（FollowJointTrajectory action）去执行，然后监控执行状态、容错、停止等
### 6. FK / IK 是怎么调用的：`compute_fk` / `compute_ik`
#### 6.1 FK：`GetPositionFK` / `compute_fk`

`compute_fk_async()`：
- 初始化 client：`srv_name="compute_fk"`
- 请求里写：
    - `fk_link_names`（默认 end_effector）
    - `robot_state.joint_state`（默认当前 joint_states）

然后 call_async,最终返回 `PoseStamped`（或多个 link 的 PoseStamped）。

**底层是谁算的 FK？**
- MoveIt 的 RobotModel / RobotState（基于 URDF + KDL/自带 kinematics）
- 本质是链式变换：base->...->link
- Python 不参与计算
#### 6.2 IK：`GetPositionIK` / `compute_ik`

`compute_ik_async()`：
- 初始化 client：`srv_name="compute_ik"`
- 写入：
    - `ik_request.group_name = self.__group_name`
    - `pose_stamped`（位置+姿态）
    - `robot_state.joint_state`（作为 seed）
    - `constraints`（可选）
    - `avoid_collisions=True`
        
然后 call_async,返回 `res.solution.joint_state`

**底层是谁算的 IK？**
- MoveIt 的 kinematics plugin（你在 `kinematics.yaml` 配的：KDL、TRAC-IK、bio_ik/pick_ik 等）
- `avoid_collisions=True` 时，MoveIt 会检查解是否碰撞（使用 planning scene + 碰撞检测器）
### 7. “从目标 Pose 到最终运动”的完整链路
#### 7.1 Python 层（moveit2.py）

1. 转 PoseStamped
2. 写 goal_constraints（PositionConstraint + OrientationConstraint）
3. 读 joint_states 写 start_state
4. 发送给 move_group（action 或 service）
    
#### 7.2 MoveIt2 核心（move_group 内部）

1. 构建 PlanningScene（机器人模型 + 当前状态 + 场景碰撞体 + ACM）
2. 处理约束（goal/path constraints）
3. 规划管线：
    - 采样规划（OMPL 等）在关节空间找一条无碰撞路径
    - 过程中大量调用：
        - FK（用来算 link 位姿做碰撞检测）
        - 约束检查
        - 碰撞检测（FCL 等）
4. 得到一条“几何路径”（joint waypoints）
5. 时间参数化（给每个点加 time_from_start，使得满足速度/加速度限制）
6. 生成 JointTrajectory
    
#### 7.3 执行（MoveIt trajectory execution）

1. 把 JointTrajectory 发给控制器（ros2_control joint_trajectory_controller）
2. 控制器插值跟踪，驱动仿真/真实关节
3. MoveIt 监控是否到达、是否超时、是否被停止

# moveit2.py API详细说明
## 1. 枚举：MoveIt2State

### `class MoveIt2State(Enum)`

用于表示当前执行状态：
- `IDLE = 0`：没有动作在请求/执行
- `REQUESTING = 1`：goal 已发出、还没被 action server 接受
- `EXECUTING = 2`：goal 已接受，正在执行（或 move_group 正在规划+执行）
    
---

## 2. 主类：MoveIt2

### 2.1 构造函数 `__init__(...)`

```
def __init__(
  node: Node,
  joint_names: List[str],
  base_link_name: str,
  end_effector_name: str,
  group_name: str = "arm",
  execute_via_moveit: bool = False,                 # deprecated
  ignore_new_calls_while_executing: bool = False,
  callback_group: Optional[CallbackGroup] = None,
  follow_joint_trajectory_action_name: str = "DEPRECATED",  # deprecated
  use_move_group_action: bool = False,
)

```

### 2.2参数含义
- `node`：rclpy Node，用于创建订阅/服务/Action client
- `joint_names`：机械臂关节名称列表（用来过滤 joint_states & 构造 JointState）
- `base_link_name`：规划参考系默认 frame
- `end_effector_name`：默认目标 link（通常是末端执行器 link）
- `group_name`：MoveIt planning group 名称（kinematics/规划用）
- `execute_via_moveit`（废弃）：旧版布尔，等价于 `use_move_group_action=True`
- `ignore_new_calls_while_executing`：如果在执行中再次发请求，是否直接忽略
- `callback_group`：让订阅/服务/action 使用同一个 callback_group（多线程/互斥用）
- `follow_joint_trajectory_action_name`（废弃）
- `use_move_group_action`：是否使用 `move_action`（MoveGroup action）进行“规划+执行一体化”
    - True：`move_to_pose/move_to_configuration` 会发 `MoveGroup` action
    - False：先 call planning service，再用 `execute_trajectory` action 执行

### 2.3初始化中创建的通信对象

#### 2.3.1Publishers
- `__collision_object_publisher`：发布 `CollisionObject` 到 `/collision_object`
- `__attached_collision_object_publisher`：发布 `AttachedCollisionObject` 到 `/attached_collision_object`
- `__cancellation_pub`：发布 `String("stop")` 到 `/trajectory_execution_event`（用于停止执行）

#### 2.3.2JointState subscriber

订阅 `joint_states`，回调 `__joint_state_callback`  
QoS：best effort，depth=1（低延迟）

#### 2.3.3Action clients

- `__move_action_client`：`MoveGroup` action，action_name=`"move_action"`
- `_execute_trajectory_action_client`：`ExecuteTrajectory` action，action_name=`"execute_trajectory"`
    
#### 2.3.4Service clients

- `_plan_kinematic_path_service`：`GetMotionPlan`，srv_name=`"plan_kinematic_path"`
- `_plan_cartesian_path_service`：`GetCartesianPath`，srv_name=`"compute_cartesian_path"`
- `_get_planning_scene_service`：`GetPlanningScene`，srv_name=`"get_planning_scene"`
- `_apply_planning_scene_service`：`ApplyPlanningScene`，srv_name=`"apply_planning_scene"`

#### 2.3.5内部状态变量

- `__move_action_goal`：通过 `__init_move_action_goal()` 生成的 `MoveGroup.Goal` 模板，请求时不断往里填约束/起点
- `__joint_state`：当前 joint state（线程锁保护）
- `__is_motion_requested/__is_executing`：用于状态机
- `motion_suceeded`：最后一次 motion 是否成功
- `__last_error_code`：最后一次返回的 `MoveItErrorCodes`
- `__execution_goal_handle`：Action goal handle
- `__future_done_event`：用于同步等待（某些内部回调）
    
---

## 3. 执行状态与控制 API

### 3.1`query_state() -> MoveIt2State`

读取内部状态，返回 `REQUESTING/EXECUTING/IDLE`。

### 3.2`cancel_execution()`

如果当前不在 EXECUTING，会 warn。  
否则向 `/trajectory_execution_event` 发布 `"stop"`。  
（具体是否能停止，要看你的 MoveIt 执行器是否订阅并实现了此事件。）

### 3.3`get_execution_future() -> Optional[Future]`

仅当 EXECUTING 时有效：返回 `goal_handle.get_result_async()`，你可以自己 await 或 add_done_callback。

### 3.4`get_last_execution_error_code() -> Optional[MoveItErrorCodes]`

返回上一次动作结束时的 MoveIt 错误码（可能为空）。

---

## 4. 高层动作 API

### 4.1 `move_to_pose(...)`

用途：输入目标位姿（Pose / PoseStamped / position+quaternion），**规划并执行**

关键参数：
- `pose` / `position` / `quat_xyzw`：三选一组合
- `target_link`：目标 link（默认 end_effector）
- `frame_id`：目标所在坐标系（默认 base_link）
- `tolerance_position / tolerance_orientation`：位置/姿态容差
- `weight_position / weight_orientation`：约束权重（MoveIt 约束 solver 会用到）
- `cartesian`：True 走笛卡尔插值服务；False 走普通运动规划
- `cartesian_max_step`：笛卡尔插值步长
- `cartesian_fraction_threshold`：笛卡尔规划完成比例阈值（不足则视为失败）
    
执行逻辑：
1. 统一成 `PoseStamped`
2. 如果 `__use_move_group_action and not cartesian`：
    - 调 `set_pose_goal(...)` 写 goal constraints
    - 把当前 joint_state 写入 `start_state`
    - `_send_goal_async_move_action()` 发 MoveGroup action（move_group 内部规划+执行）
    - 清理约束：`clear_goal_constraints()` & `clear_path_constraints()`
3. 否则：
    - `traj = plan(...)`
    - `execute(traj)`
        
---

### 4.2 `move_to_configuration(joint_positions, ...)`

用途：输入关节角目标，规划并执行

参数：
- `joint_positions`：目标关节角列表
- `joint_names`：可选，如果提供则按该顺序映射；否则按构造时的 `self.__joint_names`
- `tolerance`：每个 joint 的容差（上下同值）
- `weight`：约束权重
    
执行逻辑与 move_to_pose 类似：
- use_move_group_action=True：用 MoveGroup action 一体化
- 否则：`plan()` + `execute()`
    
---

## 5. 规划 API

### 5.1 `plan(...) -> Optional[JointTrajectory]`

同步版本：内部调用 `plan_async()`，循环等待 future.done()，然后用 `get_trajectory()` 提取 `JointTrajectory`。

参数（重点）：
- pose/position/quat_xyzw：目标位姿
- joint_positions/joint_names：目标关节
- tolerance_* 与 weight_*：约束参数
- start_joint_state：指定规划起点（JointState 或 List[float]）
- cartesian/max_step/cartesian_fraction_threshold：笛卡尔规划参数
    
返回：
- 成功：`JointTrajectory`
- 失败：`None`
    
### 5.2 `plan_async(...) -> Optional[Future]`

异步规划请求

工作流程：
1. 把输入目标写入 `__move_action_goal.request.goal_constraints`
    - pose -> set_position_goal + set_orientation_goal
    - joint_positions -> set_joint_goal
2. 设置起点 `start_state`
3. cartesian=True：调用 `_plan_cartesian_path(max_step, frame_id)`  
    否则：调用 `_plan_kinematic_path()`
4. 清理约束（避免污染下次请求）
5. 返回 Future
    
注意：这里返回的 Future 类型取决于你调用的服务（GetMotionPlan 或 GetCartesianPath）。

### 5.3 `get_trajectory(future, cartesian=False, cartesian_fraction_threshold=0.0) -> Optional[JointTrajectory]`

从 future 的结果里解析 JointTrajectory

- cartesian=True：
    - `res.error_code` 成功 且 `res.fraction >= threshold` 才返回 `res.solution.joint_trajectory`
- cartesian=False：
    - 取 `res.motion_plan_response`
    - 成功则返回 `res.trajectory.joint_trajectory`

---

## 6. 执行 API

### 6.1`execute(joint_trajectory: JointTrajectory)`

把一条 `JointTrajectory` 封装成 `ExecuteTrajectory.Goal`，通过 `execute_trajectory` action 执行

- 若正在执行且 `ignore_new_calls_while_executing=True` → 忽略
- 若传入轨迹为空 → warn
- 实际 action 发送由 `_send_goal_async_execute_trajectory()` 完成
    
### 6.2`wait_until_executed() -> bool`

阻塞等待直到：
- `__is_motion_requested` 和 `__is_executing` 都为 False  
    返回 `motion_suceeded`。
    
### 6.3`reset_controller(joint_state, sync=True)`

发送一个“dummy trajectory”（单点轨迹）把控制器状态重置到指定 joint_state。  
常用于仿真里瞬移关节（或将 controller 对齐到某状态）。

---

## 7. 目标约束 API（Goal Constraints）

### 7.1 `set_pose_goal(...)`

组合 API：等价于同时调用 `set_position_goal()` + `set_orientation_goal()`

### 7.2 PositionConstraint

#### `create_position_constraint(position, frame_id=None, target_link=None, tolerance=0.001, weight=1.0) -> PositionConstraint`

- reference frame：默认 base_link
- link：默认 end_effector
- constraint_region：一个 Sphere（半径=tolerance），中心=position
- weight：约束权重
    

#### `set_position_goal(...)`

创建 PositionConstraint，并 append 到：  
`__move_action_goal.request.goal_constraints[-1].position_constraints`

### 7.3 OrientationConstraint

#### `create_orientation_constraint(quat_xyzw, frame_id=None, target_link=None, tolerance=..., weight=1.0, parameterization=0) -> OrientationConstraint`

- reference frame：默认 base_link
- link：默认 end_effector
- orientation：目标四元数
- tolerance：若是 float → xyz 同值；若 tuple → 分轴
- parameterization：0 欧拉角 / 1 旋转向量（用于解释容差方式）
- weight：约束权重
    
#### `set_orientation_goal(...)`

创建 OrientationConstraint，并 append 到：  
`__move_action_goal.request.goal_constraints[-1].orientation_constraints`

### 7.4 JointConstraint

#### `create_joint_constraints(joint_positions, joint_names=None, tolerance=0.001, weight=1.0) -> List[JointConstraint]`

- joint_names=None：用构造时 `self.__joint_names`
- 对每个 joint：
    - `position = joint_positions[i]`
    - `tolerance_above/below = tolerance`
    - `weight = weight`

#### `set_joint_goal(...)`

把 `create_joint_constraints()` 的结果 extend 到：  
`__move_action_goal.request.goal_constraints[-1].joint_constraints`

### 7.5 约束集合管理

#### `clear_goal_constraints()`

把 `goal_constraints` 重置为 `[Constraints()]`（只留一个空 constraints）

#### `create_new_goal_constraint()`

再 append 一个新的 `Constraints()`，用于一次 request 里设置多个 goal constraints（MoveIt 会按其语义处理）。

---

## 8. 路径约束 API（Path Constraints）

这些约束在 `MotionPlanRequest.path_constraints` 里，语义是：**路径上的每个状态都应满足（或尽量满足）**

### `set_path_joint_constraint(...)`

创建 joint constraints 并 extend 到 `request.path_constraints.joint_constraints`

### `set_path_position_constraint(...)`

创建 PositionConstraint 并 append 到 `request.path_constraints.position_constraints`

### `set_path_orientation_constraint(...)`

创建 OrientationConstraint 并 append 到 `request.path_constraints.orientation_constraints`

### `clear_path_constraints()`

把 `request.path_constraints` 重置为 `Constraints()`

---

## 9. FK / IK API

### 9.1 FK

#### `compute_fk(joint_state=None, fk_link_names=None)`

同步：调用 `compute_fk_async()`，等待 future.done()，再解析。

#### `compute_fk_async(joint_state=None, fk_link_names=None) -> Optional[Future]`

- lazy init：`__init_compute_fk()`（创建 client & request）
- fk_link_names=None：默认 `[end_effector_name]`
- joint_state=None：默认使用订阅到的当前 joint_state
- service ready 检查：`service_is_ready()`
- `call_async(req)`
    

#### `get_compute_fk_result(future, fk_link_names=None)`

- 成功：返回 `PoseStamped` 或 `List[PoseStamped]`
- 失败：warn + None
    
### 9.2 IK

#### `compute_ik(position, quat_xyzw, start_joint_state=None, constraints=None, wait_for_server_timeout_sec=1.0)`

同步：调用 `compute_ik_async()` 并等待。

#### `compute_ik_async(...) -> Optional[Future]`

- lazy init：`__init_compute_ik()`
- 写入 pose_stamped pose
- start_joint_state：作为 IK seed
- constraints：作为 IK 的额外约束（可为空）
- `avoid_collisions=True`（在 init 里固定设置）
- 等待服务：`wait_for_service(timeout_sec=...)`
- `call_async(req)`
    
#### `get_compute_ik_result(future) -> Optional[JointState]`

成功返回 `res.solution.joint_state`，失败 warn。

---

## 10. JointState 管理

### `reset_new_joint_state_checker()`

把 `__new_joint_state_available=False`

### `force_reset_executing_state()`

强制把 `__is_motion_requested/__is_executing` 清零（仅用于极少数状态卡死场景）

### `__joint_state_callback(msg)`

仅当 msg 中包含 `self.joint_names` 的所有关节名才更新：
- 更新 `__joint_state`
- `__new_joint_state_available=True`
    
---

## 11. Planning Scene / 碰撞体 API

### 11.1 添加 collision primitive / box / sphere / cylinder / cone

#### `add_collision_primitive(id, primitive_type, dimensions, pose/position+quat, frame_id=None, operation=ADD)`

发布 `CollisionObject` 到 `/collision_object`：
- `header=pose_stamped.header`
- `id=id`
- `operation=ADD/MOVE/REMOVE`
- `pose=pose_stamped.pose`
- `primitives=[SolidPrimitive(...)]`
    
#### `add_collision_box(id, size, ...)`

封装调用 `add_collision_primitive(... SolidPrimitive.BOX ...)`

#### `add_collision_sphere(id, radius, ...)`

封装调用 `add_collision_primitive(... SolidPrimitive.SPHERE ...)`

#### `add_collision_cylinder(id, height, radius, ...)`

封装调用 `add_collision_primitive(... SolidPrimitive.CYLINDER ...)`

#### `add_collision_cone(id, height, radius, ...)`

封装调用 `add_collision_primitive(... SolidPrimitive.CONE ...)`

### 11.2 Mesh 碰撞体

#### `add_collision_mesh(filepath, id, pose/position+quat, frame_id=None, operation=ADD, scale=1.0, mesh=None)`

- 依赖 `trimesh`（否则抛 ImportError）
- filepath 与 mesh 必须二选一
- 支持 scale（float 或 tuple），通过 4x4 transform 缩放 mesh
- 转成 `shape_msgs/Mesh`（triangles + vertices）
- 发布到 `/collision_object`
    

### 11.3 移除与移动

#### `remove_collision_object(id)`

发布 `CollisionObject(id=id, operation=REMOVE)`

#### `remove_collision_mesh(id)`

与 `remove_collision_object` 相同

#### `move_collision(id, position, quat_xyzw, frame_id=None)`

发布 `CollisionObject(operation=MOVE)` 更新 pose

### 11.4 Attach / Detach

#### `attach_collision_object(id, link_name=None, touch_links=[], weight=0.0)`

发布 `AttachedCollisionObject` 到 `/attached_collision_object`
- link 默认 end_effector
- touch_links：允许接触的 robot links（抓取时常用）
- weight：附加负载（可用于动态）

#### `detach_collision_object(id)`

发布 remove attached object

#### `detach_all_collision_objects()`

发布 operation=REMOVE（无 id）移除所有 attached objects

---

## 12. Allowed Collision Matrix（允许碰撞矩阵）

### `update_planning_scene() -> bool`

调用 `get_planning_scene` 服务（同步 `.call()`）得到 `self.__planning_scene`

### `allow_collisions(id: str, allow: bool) -> Optional[Future]`

- 先 `update_planning_scene()`
    
- 修改 `allowed_collision_matrix`：
    - 确保 `id` 在 entry_names 中
    - 对所有 entry_values 设置与 `id` 的 allow 值
- 调 `apply_planning_scene` 服务（async）  
    返回 Future
    
### `process_allow_collision_future(future) -> bool`

- future 未完成：False
- 如果 apply 失败：还原旧 ACM
- 返回 success
    
---

## 13. 清空场景

### `clear_all_collision_objects() -> Optional[Future]`

- `update_planning_scene()`
- 备份旧 scene
- 清空：
    - `world.collision_objects = []`
    - `robot_state.attached_collision_objects = []`
- apply_planning_scene async
    
### `cancel_clear_all_collision_objects_future(future)`

取消 pending request

### `process_clear_all_collision_objects_future(future) -> bool`

apply 失败则恢复旧 scene

---

## 14. 私有规划/执行内部函数（理解底层链路最关键）

### `_plan_kinematic_path() -> Optional[Future]`

- 把 `__move_action_goal.request` 直接塞进 `GetMotionPlan.Request().motion_plan_request`
- 填 stamp（workspace & constraints headers）
- call `plan_kinematic_path` async
    
### `_plan_cartesian_path(max_step=0.0025, frame_id=None) -> Optional[Future]`

- 构造 `GetCartesianPath.Request`
- group、link、max_step、frame、stamp
- path_constraints 继承自 `__move_action_goal.request.path_constraints`
- waypoints：从 `goal_constraints[-1]` 里取最后一个 position/orientation 组合成 Pose
- call `compute_cartesian_path` async
    
### `_send_goal_async_move_action()`

- 填 stamp
- 检查 action server ready
- `send_goal_async(self.__move_action_goal)`
- future done -> `__response_callback_move_action`
    
### `__response_callback_move_action(response)`

- 若不 accepted：`__is_motion_requested=False`
- accepted：设置 goal_handle、`__is_executing=True`
- 注册 result callback：`__result_callback_move_action`
    
### `__result_callback_move_action(res)`

- 根据 `GoalStatus` 设置 `motion_suceeded`
- 记录 `__last_error_code`
- 清理执行状态
    
### `_send_goal_async_execute_trajectory(goal, wait_until_response=False)`

发 `ExecuteTrajectory` action goal，done -> `__response_callback_execute_trajectory`

### `__response_callback_execute_trajectory(response)` / `__result_callback_execute_trajectory(res)`

与 move_action 类似：管理执行状态与结果码

---

## 15. 静态方法：`__init_move_action_goal(...) -> MoveGroup.Goal`

生成一个 MoveGroup.Goal 模板，包含：
- workspace parameters：[-1,1] box
- goal_constraints 初始为 `[Constraints()]`
- path_constraints 初始空
- pipeline_id / planner_id 默认为空字符串
- group_name、attempts、allowed_planning_time 默认值
- velocity/acc scaling、cartesian speed 限制字段（兼容 Humble/Iron 字段差异）
- planning_options.plan_only=False（即会执行）

这就是后续规划/执行所有请求的“母板”。

---

## 16. FK/IK client 初始化私有函数

### `__init_compute_fk()`

创建 `GetPositionFK` client，srv_name=`compute_fk`，并初始化 request 固定字段。

### `__init_compute_ik()`

创建 `GetPositionIK` client，srv_name=`compute_ik`，初始化：
- group_name
- avoid_collisions=True
- pose header frame_id = base_link
    
---

## 17. 属性（properties）

- `planning_scene`：返回缓存的 PlanningScene（需先 update）
- `follow_joint_trajectory_action_client`：⚠️你贴的代码里这个属性返回 `self.__follow_joint_trajectory_action_client`，但该成员变量在构造中并未定义，可能是遗留/bug（你若访问会报 AttributeError）
- `end_effector_name / base_link_name / joint_names`
- `joint_state`：线程安全读取当前 joint state
- `new_joint_state_available`
- `max_velocity`：读写 `request.max_velocity_scaling_factor`
- `max_acceleration`：读写 `request.max_acceleration_scaling_factor`
- `num_planning_attempts`
- `allowed_planning_time`
    
- `cartesian_avoid_collisions / cartesian_jump_threshold / cartesian_prismatic_jump_threshold / cartesian_revolute_jump_threshold`
    
    - ⚠️代码里这些 property 使用了 `self.__cartesian_path_request.request.xxx` 这种访问方式，但 `_plan_cartesian_path()` 里用的是 `self.__cartesian_path_request.xxx`。这提示该文件可能存在版本差异/字段写错，实际运行需以消息定义为准（否则会 AttributeError）
- `pipeline_id` / `planner_id`：读写 MoveGroup request 字段
    
---

## 18. 文件级辅助函数（非类成员）

### `init_joint_state(joint_names, joint_positions=None, joint_velocities=None, joint_effort=None) -> JointState`

构造一个 `sensor_msgs/JointState`：
- name = joint_names
- position/velocity/effort 缺省时填 0
    
### `init_execute_trajectory_goal(joint_trajectory) -> Optional[ExecuteTrajectory.Goal]`

- joint_trajectory None -> None
- 否则封装到 `ExecuteTrajectory.Goal().trajectory.joint_trajectory`
    
### `init_dummy_joint_trajectory_from_state(joint_state, duration_sec=0, duration_nanosec=0) -> JointTrajectory`

构造单点轨迹（用于 reset_controller）：
- joint_names = joint_state.name
- point.positions = joint_state.position
- point.time_from_start = duration
    
---

## 19. 作为使用者的“最重要调用模式”

### A) 一体化（MoveGroup action）

`moveit2.use_move_group_action = True moveit2.move_to_pose(...) moveit2.wait_until_executed()`

内部：MoveGroup action -> move_group 规划+执行。

### B) 分离式（能拿到轨迹）

`traj = moveit2.plan(...) if traj:     moveit2.execute(traj)     moveit2.wait_until_executed()`

内部：GetMotionPlan / GetCartesianPath -> JointTrajectory -> ExecuteTrajectory action。

### C) 显式 IK/FK

`ik = moveit2.compute_ik(pos, quat, constraints=...) fk = moveit2.compute_fk(joint_state=ik)`

---

## 20. “正/逆解与运动的底层逻辑”一句话总结

- **IK**：`compute_ik` 服务 -> MoveIt kinematics plugin 计算，`avoid_collisions=True` 会检查碰撞
- **FK**：`compute_fk` 服务 -> MoveIt RobotState 计算
- **规划**：`plan_kinematic_path`（OMPL/CHOMP 等）或 `compute_cartesian_path`（插值+IK+碰撞检查）
- **执行**：`execute_trajectory` action -> MoveIt 执行管理 -> ros2_control joint_trajectory_controller