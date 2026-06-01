### 1.说明（cartesian=True时无法时间参数化）

ROS2 Humble 的 **compute_cartesian_path（GetCartesianPath）**本身不支持把 velocity/acc scaling 传进去

compute_cartesian_path返回的 JointTrajectory往往**没有正确的 **time_from_start**（所以“原样 execute”时，速度控制不由 MoveIt 的 scaling 决定

纯 Python 里没有 MoveIt 的 TOTG/IPP 时间参数化实现可直接用。 
✅ 所以：**最稳方案是：moveit2.py 内部自动调用一个 retime ROS2 service**（C++ 实现 TOTG），把轨迹 retime 后再返回/执行。  
也就是：**“封装里改 + 外部有个 retime service 节点”** 这套组合

### 2.在moveit2.py内部实现封装TOTG/IPP 时间参数化
#### 2.1在 moveit2.py 顶部新增 import
```
from rclpy.duration import Duration
from trajectory_retime_server.srv import RetimeTrajectory
```

#### 2.2在 MoveIt2.**init** 里新增配置项（参数+成员变量）

在 `class MoveIt2.__init__(...)` 的参数列表里，加这些参数
```
        retime_cartesian: bool = True,
        retime_service_name: str = "/retime_trajectory",
        retime_wait_timeout_sec: float = 0.5,
```

然后在 `__init__` 方法末尾新增成员变量：
```
  # --- Cartesian retime options ---
        self.__retime_cartesian = retime_cartesian
        self.__retime_service_name = retime_service_name
        self.__retime_wait_timeout_sec = float(retime_wait_timeout_sec)

        self.__retime_client = None
        if self.__retime_cartesian:
            self.__init_retime_client()
```

#### 2.3在 MoveIt2 类中新增一个初始化 retime client 的私有方法

在 `MoveIt2` 类里放在 `__init_compute_ik()` 旁边，新增：
```
 def __init_retime_client(self):
        """
        Initialize retime service client (optional).
        This requires an external retime server, e.g. trajectory_retime_server.
        """
        if RetimeTrajectory is None:
            self._node.get_logger().warn(
                "RetimeTrajectory service type not available. "
                "Install trajectory_retime_server package to enable cartesian retiming."
            )
            return

        self.__retime_client = self._node.create_client(
            srv_type=RetimeTrajectory,
            srv_name=self.__retime_service_name,
            callback_group=self._callback_group,
        )
```

#### 2.4在 MoveIt2 类中新增一个“retime trajectory”的方法（核心）

在 `MoveIt2` 类里新增：
```
    def retime_trajectory_if_needed(
        self,
        traj: Optional[JointTrajectory],
        cartesian: bool,
        group_name: Optional[str] = None,
    ) -> Optional[JointTrajectory]:
        """
        If cartesian=True and retime_cartesian enabled, call retime service to add proper timing
        so that max_velocity/max_acceleration scaling can take effect in Humble.
        """
        if traj is None:
            return None
        if not cartesian:
            return traj

        if not self.__retime_cartesian:
            return traj

        if RetimeTrajectory is None or self.__retime_client is None:
            # retime not available
            return traj

        # Require service ready
        if not self.__retime_client.wait_for_service(timeout_sec=self.__retime_wait_timeout_sec):
            self._node.get_logger().warn(
                f"Retime service '{self.__retime_service_name}' not available; "
                "executing raw cartesian trajectory (scaling may not take effect)."
            )
            return traj

        # Build request
        req = RetimeTrajectory.Request()
        req.trajectory = traj

        req.group_name = group_name if group_name is not None else self.__group_name

        # Use scaling factors already set via properties max_velocity/max_acceleration
        req.velocity_scaling = float(self.__move_action_goal.request.max_velocity_scaling_factor)
        req.acceleration_scaling = float(self.__move_action_goal.request.max_acceleration_scaling_factor)

        future = self.__retime_client.call_async(req)

        # Wait synchronously (keep style consistent with plan())
        rate = self._node.create_rate(200)
        while not future.done():
            rate.sleep()

        res = future.result()
        if res is None or (hasattr(res, "success") and not res.success):
            msg = getattr(res, "message", "unknown")
            self._node.get_logger().warn(f"Retime failed: {msg}. Executing raw trajectory.")
            return traj

        return res.retimed

```
这个函数做了什么：
- 只在 `cartesian=True` 时启用
- 使用已经设置的 `max_velocity/max_acceleration`
- 调 retime service 拿到带时间戳的轨迹
- 拿不到就回退到原轨迹（不中断功能）

#### 2.5修改 get_trajectory()：在 cartesian 分支返回前调用 retime

当前 `get_trajectory()` 里，cartesian 分支成功时是：
```
return res.solution.joint_trajectory
```

改成：
```
traj = res.solution.joint_trajectory
traj = self.retime_trajectory_if_needed(traj, cartesian=True)
return traj

```

#### 2.6修改 move_to_pose()/move_to_configuration() 的 else 分支

moveit2.py`MoveIt2.move_to_pose()` 的 else 分支现在是：
```
else:
            # Plan via MoveIt 2 and then execute directly with the controller
            self.execute(
                self.plan(
                    ...
                    cartesian=cartesian,
                    ...
                )
            )
```

修改成：
```
        else:
            traj = self.plan(
                position=pose_stamped.pose.position,
                quat_xyzw=pose_stamped.pose.orientation,
                frame_id=pose_stamped.header.frame_id,
                target_link=target_link,
                tolerance_position=tolerance_position,
                tolerance_orientation=tolerance_orientation,
                weight_position=weight_position,
                weight_orientation=weight_orientation,
                cartesian=cartesian,
                max_step=cartesian_max_step,
                cartesian_fraction_threshold=cartesian_fraction_threshold,
            )
            traj = self._retime_trajectory_if_needed(traj, cartesian=cartesian)
            self.execute(traj)

```

### 3.创建retime service C++节点
#### 3.1创建trajectory_retime_server功能包
##### 3.1.1创建创建 service 文件
```
ros2 pkg create trajectory_retime_server --build-type ament_cmake
cd ~/ros2_ws(工作总空间)/src/trajectory_retime_server
mkdir srv
创建 `srv/RetimeTrajectory.srv`：
trajectory_msgs/JointTrajectory trajectory
string group_name
float64 velocity_scaling
float64 acceleration_scaling
---
trajectory_msgs/JointTrajectory retimed
bool success
string message
```

##### 3.1.2修改 package.xml、CMakeLists.txt
```
<package format="3">
  <name>trajectory_retime_server</name>
  <version>0.0.0</version>
  <description>Trajectory retime service and server node</description>
  <maintainer email="jasonlee@todo.todo">jasonlee</maintainer>
  <license>Apache-2.0</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <build_depend>rosidl_default_generators</build_depend>
  <build_depend>trajectory_msgs</build_depend>
  <build_depend>moveit_msgs</build_depend>
  <exec_depend>moveit_msgs</exec_depend>
  <exec_depend>rosidl_default_runtime</exec_depend>
  <exec_depend>trajectory_msgs</exec_depend>
  <exec_depend>xacro</exec_depend>
  <exec_depend>urdf</exec_depend>

  <export>
    <build_type>ament_cmake</build_type>
    <member_of_group>rosidl_interface_packages</member_of_group>
  </export>
</package>

```

```
cmake_minimum_required(VERSION 3.8)
project(trajectory_retime_server)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(trajectory_msgs REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(moveit_msgs REQUIRED)
find_package(moveit_core REQUIRED)
find_package(moveit_ros_planning REQUIRED)
ament_package_xml()
list(APPEND "${PROJECT_NAME}_MEMBER_OF_GROUPS" "rosidl_interface_packages")
# Generate service interfaces
rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/RetimeTrajectory.srv"
  DEPENDENCIES trajectory_msgs
)
ament_export_dependencies(rosidl_default_runtime)

# ---- C++ node ----
add_executable(retime_server src/retime_server.cpp)
ament_target_dependencies(retime_server
  rclcpp
  trajectory_msgs
  moveit_core
  moveit_ros_planning
  moveit_msgs
)

# Link generated service typesupport
rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME} "rosidl_typesupport_cpp")
target_link_libraries(retime_server "${cpp_typesupport_target}")

install(TARGETS retime_server
  DESTINATION lib/${PROJECT_NAME}
)
install(DIRECTORY launch
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

##### 3.1.3创建retime_server.cpp文件

在 `trajectory_retime_server/src/retime_server.cpp` 创建

```
#include <memory>
#include <string>
#include <unordered_map>
#include <vector>
#include <chrono>
#include "rclcpp/rclcpp.hpp"
#include "rclcpp/parameter_client.hpp"
#include "trajectory_msgs/msg/joint_trajectory.hpp"
#include "moveit_msgs/msg/robot_trajectory.hpp"
#include "moveit/robot_model_loader/robot_model_loader.h"
#include "moveit/robot_state/robot_state.h"
#include "moveit/robot_trajectory/robot_trajectory.h"
#include "moveit/trajectory_processing/time_optimal_trajectory_generation.h"
#include "trajectory_retime_server/srv/retime_trajectory.hpp"

/*
提供一个 ROS2 Service：/retime_trajectory
输入：一条 JointTrajectory + group_name + 速度/加速度缩放
输出：用 MoveIt 的 TOTG（TimeOptimalTrajectoryGeneration） 给这条轨迹重新计算 time_from_start（以及可选的 velocities/accelerations），生成“符合关节限位 + 尽可能快”的时间戳轨迹。
*/

namespace
{
// 把输入限制到 [0, 1] 区间，防止用户传入非法 scaling
double clamp01(double v)
{
  if (v < 0.0) return 0.0;
  if (v > 1.0) return 1.0;
  return v;
}

// ROS2 Duration 转 double 秒（用于返回 message 里的 total_time）
double toSec(const builtin_interfaces::msg::Duration & d)
{
  return static_cast<double>(d.sec) + static_cast<double>(d.nanosec) * 1e-9;
}

// 等待某个节点的 parameter service 可用（这里用于 /move_group）
bool wait_for_param_service(
  const std::shared_ptr<rclcpp::SyncParametersClient>& client,
  rclcpp::Logger logger,
  double timeout_sec)
{
  const auto start = std::chrono::steady_clock::now();

  while (!client->wait_for_service(std::chrono::milliseconds(200))) {
    RCLCPP_WARN(logger, "Waiting for parameter service...");

    const double elapsed =
      std::chrono::duration<double>(std::chrono::steady_clock::now() - start).count();

    if (elapsed > timeout_sec) {
      return false;  // 超时退出
    }
  }
  return true;
}
}  // namespace


// -----------------------------
// TrajectoryRetimeServer：核心类
// 作用：提供 /retime_trajectory 服务，对输入 JointTrajectory 进行 TOTG 时间参数化
// -----------------------------
class TrajectoryRetimeServer
{
public:
  explicit TrajectoryRetimeServer(const rclcpp::Node::SharedPtr& node)
  : node_(node)
  {
    // 1) 创建服务：/retime_trajectory
    // 服务类型：trajectory_retime_server::srv::RetimeTrajectory
    // 回调函数：handle()
    srv_ = node_->create_service<trajectory_retime_server::srv::RetimeTrajectory>(
      "/retime_trajectory",
      std::bind(&TrajectoryRetimeServer::handle, this,
                std::placeholders::_1, std::placeholders::_2));

    // 2) 尝试从 /move_group 节点获取 robot_description（URDF）
    if (!ensure_robot_description_from_move_group()) {
      RCLCPP_ERROR(node_->get_logger(),
        "Failed to get 'robot_description' from /move_group. "
        "Make sure move_group is running and provides robot_description parameter.");
    }

    // 3) 从本节点的 robot_description 参数加载 RobotModel
    load_robot_model();
  }

private:
  // -----------------------------
  // 从 /move_group 拉取 robot_description，并 set 到本节点
  // -----------------------------
  bool ensure_robot_description_from_move_group()
  {
    // (A) 如果本节点已经有 robot_description 且非空，则直接使用，不再拉取
    if (node_->has_parameter("robot_description")) {
      auto p = node_->get_parameter("robot_description");
      if (p.get_type() == rclcpp::ParameterType::PARAMETER_STRING &&
          !p.as_string().empty()) {
        RCLCPP_INFO(node_->get_logger(), "robot_description already exists on this node.");
        return true;
      }
    }

    // (B) 如果未声明该参数，先声明
    if (!node_->has_parameter("robot_description")) {
      node_->declare_parameter<std::string>("robot_description", "");
    }

    // (C) 创建同步参数客户端，目标节点 /move_group
    auto client = std::make_shared<rclcpp::SyncParametersClient>(node_, "/move_group");

    // (D) 等待 /move_group 的 parameter service ready
    if (!wait_for_param_service(client, node_->get_logger(), 3.0)) {
      RCLCPP_ERROR(node_->get_logger(), "Parameter service of /move_group not available.");
      return false;
    }

    // (E) 获取参数 robot_description
    std::vector<rclcpp::Parameter> params;
    try {
      params = client->get_parameters({"robot_description"});
    } catch (const std::exception& e) {
      RCLCPP_ERROR(node_->get_logger(),
                   "Exception when getting parameters from /move_group: %s", e.what());
      return false;
    }

    // (F) 校验返回参数类型与内容
    if (params.empty() || params[0].get_type() != rclcpp::ParameterType::PARAMETER_STRING) {
      RCLCPP_ERROR(node_->get_logger(),
                   "/move_group does not provide robot_description as string parameter.");
      return false;
    }

    const std::string urdf = params[0].as_string();
    if (urdf.empty()) {
      RCLCPP_ERROR(node_->get_logger(), "/move_group robot_description is empty.");
      return false;
    }

    // (G) 将 URDF 设置到本节点参数中，供 RobotModelLoader 使用
    node_->set_parameter(rclcpp::Parameter("robot_description", urdf));
    RCLCPP_INFO(node_->get_logger(), "Fetched robot_description from /move_group and set it on this node.");
    return true;
  }

  // -----------------------------
  // 从 robot_description 加载 MoveIt RobotModel
  // RobotModel 里包含：
  // - 机器人关节/连杆结构（URDF）
  // - SRDF group
  // - 关节限位（速度/加速度等）
  // TOTG retime 就依赖这些限位信息
  // -----------------------------
  void load_robot_model()
  {
    // RobotModelLoader 会从 node_ 的参数 "robot_description" 读取 URDF
    robot_model_loader_ =
      std::make_shared<robot_model_loader::RobotModelLoader>(node_, "robot_description");

    robot_model_ = robot_model_loader_->getModel();
    if (!robot_model_) {
      RCLCPP_ERROR(node_->get_logger(),
        "Failed to load RobotModel from 'robot_description'. "
        "Either robot_description is missing on this node, or the URDF is invalid.");
    } else {
      RCLCPP_INFO(node_->get_logger(), "Loaded robot model: %s", robot_model_->getName().c_str());
    }
  }

  // -----------------------------
  // Service 回调：核心 retime 逻辑
  // 输入：req->trajectory (JointTrajectory) + req->group_name + scaling
  // 输出：res->retimed (JointTrajectory) + success/message
  // -----------------------------
  void handle(
    const std::shared_ptr<trajectory_retime_server::srv::RetimeTrajectory::Request> req,
    std::shared_ptr<trajectory_retime_server::srv::RetimeTrajectory::Response> res)
  {
    // 先初始化 response
    res->success = false;
    res->message.clear();
    res->retimed = trajectory_msgs::msg::JointTrajectory();

    // (1) RobotModel 必须已加载
    if (!robot_model_) {
      res->message = "RobotModel not loaded (robot_description missing on this node).";
      return;
    }

    // (2) 输入轨迹检查
    const auto & in = req->trajectory;
    if (in.joint_names.empty() || in.points.empty()) {
      res->message = "Input trajectory is empty.";
      return;
    }

    // (3) 获取 group_name 对应的 JointModelGroup
    // group_name 来自 SRDF（MoveIt config）
    const std::string group_name = req->group_name;
    const moveit::core::JointModelGroup * jmg = robot_model_->getJointModelGroup(group_name);
    if (!jmg) {
      res->message = "JointModelGroup not found: " + group_name;
      return;
    }

    // (4) 速度/加速度缩放 factor（限制在 [0,1]）
    // TOTG 内部如果传 0，可能产生数值/边界问题，所以用很小值替代
    double vel_scale = clamp01(req->velocity_scaling);
    double acc_scale = clamp01(req->acceleration_scaling);
    if (vel_scale <= 0.0) vel_scale = 1e-3;
    if (acc_scale <= 0.0) acc_scale = 1e-3;

    // (5) 建立输入 joint_names 的 name->index 映射
    // 因为输入 joint 顺序可能与 MoveIt group 内部顺序不同
    std::unordered_map<std::string, size_t> name_to_idx;
    name_to_idx.reserve(in.joint_names.size());
    for (size_t i = 0; i < in.joint_names.size(); ++i) {
      name_to_idx[in.joint_names[i]] = i;
    }

    // (6) 确保输入轨迹包含该 group 需要的所有关节
    const std::vector<std::string> group_joint_names = jmg->getVariableNames();
    for (const auto & jn : group_joint_names) {
      if (name_to_idx.find(jn) == name_to_idx.end()) {
        res->message = "Input trajectory missing joint required by group '" + group_name + "': " + jn;
        return;
      }
    }

    // (7) 构建 MoveIt RobotTrajectory（只用 positions 作为路径）
    // 注意：这里不直接使用输入的 time_from_start，
    // 因为我们要让 TOTG 重新计算最优时间戳。
    moveit::core::RobotState state(robot_model_);
    state.setToDefaultValues();
    state.update();

    robot_trajectory::RobotTrajectory rt(robot_model_, jmg);

    // nominal_dt 只是“占位”，用于让 RobotTrajectory 接受 waypoint
    // TOTG 最终会覆盖 timing
    constexpr double nominal_dt = 0.01;

    for (size_t pi = 0; pi < in.points.size(); ++pi) {
      const auto & pt = in.points[pi];

      // positions 数量必须与输入 joint_names 数量一致
      if (pt.positions.size() != in.joint_names.size()) {
        res->message = "Point positions size does not match joint_names size.";
        return;
      }

      // 把输入 pt.positions 写入 state（按照 joint name 映射）
      for (const auto & jn : group_joint_names) {
        const size_t idx = name_to_idx[jn];
        state.setVariablePosition(jn, pt.positions[idx]);
      }
      state.update();

      // 把 waypoint 加到轨迹中
      rt.addSuffixWayPoint(state, (pi == 0) ? 0.0 : nominal_dt);
    }

    // (8) 调用 TOTG：根据关节限位 + scaling 重新计算 time_from_start（以及速度/加速度）
    trajectory_processing::TimeOptimalTrajectoryGeneration totg;
    const bool ok = totg.computeTimeStamps(rt, vel_scale, acc_scale);
    if (!ok) {
      res->message = "TOTG computeTimeStamps() failed.";
      return;
    }

    // (9) 导出轨迹为 JointTrajectory
    moveit_msgs::msg::RobotTrajectory out_msg;
    rt.getRobotTrajectoryMsg(out_msg);

    trajectory_msgs::msg::JointTrajectory out = out_msg.joint_trajectory;

    // (10) 输出 joint 顺序可能是 MoveIt group 内部顺序，
    // 为了保证下游按输入顺序处理，这里把 out 重排成 in.joint_names 的顺序
    std::unordered_map<std::string, size_t> out_name_to_idx;
    out_name_to_idx.reserve(out.joint_names.size());
    for (size_t i = 0; i < out.joint_names.size(); ++i) {
      out_name_to_idx[out.joint_names[i]] = i;
    }

    // 确保输出包含输入所需的所有 joints（理论上应该都有）
    for (const auto & jn : in.joint_names) {
      if (out_name_to_idx.find(jn) == out_name_to_idx.end()) {
        res->message = "Retime output missing joint: " + jn;
        return;
      }
    }

    // 对每个点重排 positions/velocities/accelerations
    for (auto & pt : out.points) {
      std::vector<double> new_pos(in.joint_names.size(), 0.0);

      const bool has_vel = !pt.velocities.empty();
      const bool has_acc = !pt.accelerations.empty();
      std::vector<double> new_vel;
      std::vector<double> new_acc;
      if (has_vel) new_vel.resize(in.joint_names.size(), 0.0);
      if (has_acc) new_acc.resize(in.joint_names.size(), 0.0);

      for (size_t i = 0; i < in.joint_names.size(); ++i) {
        const auto & jn = in.joint_names[i];
        const size_t src = out_name_to_idx[jn];

        new_pos[i] = pt.positions[src];
        if (has_vel) new_vel[i] = pt.velocities[src];
        if (has_acc) new_acc[i] = pt.accelerations[src];
      }

      pt.positions = std::move(new_pos);
      if (has_vel) pt.velocities = std::move(new_vel);
      if (has_acc) pt.accelerations = std::move(new_acc);
    }

    out.joint_names = in.joint_names;

    // (11) 返回
    res->retimed = out;
    res->success = true;

    // (12) 返回信息中带上总时间，便于你调试 scaling 是否生效
    if (!out.points.empty()) {
      const double tN = toSec(out.points.back().time_from_start);
      res->message = "OK, total_time_sec=" + std::to_string(tN) +
                     ", vel_scale=" + std::to_string(vel_scale) +
                     ", acc_scale=" + std::to_string(acc_scale);
    } else {
      res->message = "OK";
    }
  }

private:
  rclcpp::Node::SharedPtr node_;
  rclcpp::Service<trajectory_retime_server::srv::RetimeTrajectory>::SharedPtr srv_;

  std::shared_ptr<robot_model_loader::RobotModelLoader> robot_model_loader_;
  moveit::core::RobotModelPtr robot_model_;
};


// -----------------------------
// main：创建节点并 spin
// -----------------------------
int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);

  // automatically_declare_parameters_from_overrides(true)
  // 作用：允许你在 launch / 命令行里覆盖参数时，不必提前 declare（更方便）
  rclcpp::NodeOptions options;
  options.automatically_declare_parameters_from_overrides(true);

  // 注意：如果你在同一进程里创建多个同名 node，就可能触发你看到的 rosout warning
  auto node = std::make_shared<rclcpp::Node>("trajectory_retime_server", options);

  auto server = std::make_shared<TrajectoryRetimeServer>(node);

  rclcpp::spin(node);
  rclcpp::shutdown();
  return 0;
}
```

**作用：**
**提供一个 ROS2 Service：/retime_trajectory**
**输入：一条 JointTrajectory + group_name + 速度/加速度缩放**
**输出：用 MoveIt 的 TOTG（TimeOptimalTrajectoryGeneration） 给这条轨迹重新计算 time_from_start（以及可选的 velocities/accelerations），生成“符合关节限位 + 尽可能快”的时间戳轨迹**

##### 3.1.4创建retime_server.launch.py文件

```
#!/usr/bin/env python3
import os

from launch import LaunchDescription
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory

import xacro


def generate_launch_description():
    # 这里指向你 MoveIt 配置里的 xacro
    xacro_file = os.path.join(
        get_package_share_directory("s622_moveit_config"),
        "config",
        "s622_moveit_descriptions.urdf.xacro",
    )

    if not os.path.exists(xacro_file):
        raise RuntimeError(f"Xacro file not found: {xacro_file}")

    doc = xacro.process_file(xacro_file)
    robot_description = {"robot_description": doc.toxml()}

    retime_server = Node(
        package="trajectory_retime_server",
        executable="retime_server",
        name="trajectory_retime_server",
        output="screen",
        parameters=[robot_description],
    )

    return LaunchDescription([retime_server])
```

##### 3.1.5编译
```
colcon build --symlink-install
source install/setup.bash
```