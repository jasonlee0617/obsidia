```
#include <rclcpp/rclcpp.hpp>
#include <moveit/planning_scene/planning_scene.h>
#include <moveit/planning_scene_interface/planning_scene_interface.h>
#include <moveit/task_constructor/task.h>
#include <moveit/task_constructor/solvers.h>
#include <moveit/task_constructor/stages.h>
#if __has_include(<tf2_geometry_msgs/tf2_geometry_msgs.hpp>)
#include <tf2_geometry_msgs/tf2_geometry_msgs.hpp>
#else
#include <tf2_geometry_msgs/tf2_geometry_msgs.h>
#endif
#if __has_include(<tf2_eigen/tf2_eigen.hpp>)
#include <tf2_eigen/tf2_eigen.hpp>
#else
#include <tf2_eigen/tf2_eigen.h>
#endif

static const rclcpp::Logger LOGGER = rclcpp::get_logger("mtc_tutorial");
namespace mtc = moveit::task_constructor;

class MTCTaskNode
{
public:
  MTCTaskNode(const rclcpp::NodeOptions& options);

  rclcpp::node_interfaces::NodeBaseInterface::SharedPtr getNodeBaseInterface();

  void doTask();

  void setupPlanningScene();

private:
  // Compose an MTC task from a series of stages.
  mtc::Task createTask();
  mtc::Task task_;
  rclcpp::Node::SharedPtr node_;
};

rclcpp::node_interfaces::NodeBaseInterface::SharedPtr MTCTaskNode::getNodeBaseInterface()
{
  return node_->get_node_base_interface();
}

MTCTaskNode::MTCTaskNode(const rclcpp::NodeOptions& options)
  : node_{ std::make_shared<rclcpp::Node>("mtc_node", options) }
{
}

void MTCTaskNode::setupPlanningScene()
{
  moveit_msgs::msg::CollisionObject object;
  object.id = "object";
  object.header.frame_id = "world";
  object.primitives.resize(1);
  object.primitives[0].type = shape_msgs::msg::SolidPrimitive::CYLINDER;
  object.primitives[0].dimensions = { 0.04, 0.01 };

  geometry_msgs::msg::Pose pose;
  pose.position.x = 0.15;
  pose.position.y = 0.25;
  pose.orientation.w = 1.0;
  object.pose = pose;

  moveit::planning_interface::PlanningSceneInterface psi;
  psi.applyCollisionObject(object);
}

void MTCTaskNode::doTask()
{
  task_ = createTask();

  try
  {
    task_.init();
  }
  catch (mtc::InitStageException& e)
  {
    RCLCPP_ERROR_STREAM(LOGGER, e);
    return;
  }

  if (!task_.plan(5 /* max_solutions */))
  {
    RCLCPP_ERROR_STREAM(LOGGER, "Task planning failed");
    return;
  }
  task_.introspection().publishSolution(*task_.solutions().front());

  auto result = task_.execute(*task_.solutions().front());
  if (result.val != moveit_msgs::msg::MoveItErrorCodes::SUCCESS)
  {
    RCLCPP_ERROR_STREAM(LOGGER, "Task execution failed");
    return;
  }

  return;
}

mtc::Task MTCTaskNode::createTask()
{
  mtc::Task task;
  task.stages()->setName("demo task");
  task.loadRobotModel(node_);

  const auto& arm_group_name = "dummy2_arm";
  const auto& hand_group_name = "hand";
  const auto& hand_frame = "grasp_frame";
  // const auto& hand_frame = "link6_1";

  // Set task properties
  task.setProperty("group", arm_group_name);
  task.setProperty("eef", hand_group_name);
  task.setProperty("ik_frame", hand_frame);

  mtc::Stage* current_state_ptr = nullptr;  // Forward current_state on to grasp pose generator
  auto stage_state_current = std::make_unique<mtc::stages::CurrentState>("current");
  current_state_ptr = stage_state_current.get();
  task.add(std::move(stage_state_current));

  auto sampling_planner = std::make_shared<mtc::solvers::PipelinePlanner>(node_);
  auto interpolation_planner = std::make_shared<mtc::solvers::JointInterpolationPlanner>();

  auto cartesian_planner = std::make_shared<mtc::solvers::CartesianPath>();
  cartesian_planner->setMaxVelocityScalingFactor(1.0);
  cartesian_planner->setMaxAccelerationScalingFactor(1.0);
  cartesian_planner->setStepSize(.01);

  
  auto stage_open_hand =
      std::make_unique<mtc::stages::MoveTo>("open hand", interpolation_planner);
  
  stage_open_hand->setGroup(hand_group_name);
  stage_open_hand->setGoal("open");
  task.add(std::move(stage_open_hand));

  
  auto stage_move_to_pick = std::make_unique<mtc::stages::Connect>(
      "move to pick",
      mtc::stages::Connect::GroupPlannerVector{ { arm_group_name, sampling_planner } });
  
  stage_move_to_pick->setTimeout(5.0);
  stage_move_to_pick->properties().configureInitFrom(mtc::Stage::PARENT);
  task.add(std::move(stage_move_to_pick));

  // clang-format off
  mtc::Stage* attach_object_stage =
      nullptr;  // Forward attach_object_stage to place pose generator
  // clang-format on

  // This is an example of SerialContainer usage. It's not strictly needed here.
  // In fact, `task` itself is a SerialContainer by default.
  {
    auto grasp = std::make_unique<mtc::SerialContainer>("pick object");
    task.properties().exposeTo(grasp->properties(), { "eef", "group", "ik_frame" });
    // clang-format off
    grasp->properties().configureInitFrom(mtc::Stage::PARENT,
                                          { "eef", "group", "ik_frame" });
    // clang-format on

    {
      // clang-format off
      auto stage =
          std::make_unique<mtc::stages::MoveRelative>("approach object", cartesian_planner);
      // clang-format on
      stage->properties().set("marker_ns", "approach_object");
      stage->properties().set("link", hand_frame);
      stage->properties().configureInitFrom(mtc::Stage::PARENT, { "group" });
      stage->setMinMaxDistance(0.01, 0.1);

      // Set hand forward direction
      geometry_msgs::msg::Vector3Stamped vec;
      vec.header.frame_id = hand_frame;
      vec.vector.y = 1.0;
      stage->setDirection(vec);
      grasp->insert(std::move(stage));
    }

    /****************************************************
  ---- *               Generate Grasp Pose                *
     ***************************************************/
    {
      // Sample grasp pose
      auto stage = std::make_unique<mtc::stages::GenerateGraspPose>("generate grasp pose");
      stage->properties().configureInitFrom(mtc::Stage::PARENT);
      stage->properties().set("marker_ns", "grasp_pose");
      stage->setPreGraspPose("open");
      stage->setObject("object");
      stage->setAngleDelta(M_PI / 12);
      stage->setMonitoredStage(current_state_ptr);  // Hook into current state

      // This is the transform from the object frame to the end-effector frame
      Eigen::Isometry3d grasp_frame_transform;
      Eigen::Quaterniond q = Eigen::AngleAxisd(0, Eigen::Vector3d::UnitX()) *
                             Eigen::AngleAxisd(0, Eigen::Vector3d::UnitY()) *
                             Eigen::AngleAxisd(0, Eigen::Vector3d::UnitZ());
      grasp_frame_transform.linear() = q.matrix();
      grasp_frame_transform.translation().y() = 0.03;

      // Compute IK
      // clang-format off
      auto wrapper =
          std::make_unique<mtc::stages::ComputeIK>("grasp pose IK", std::move(stage));
      // clang-format on
      wrapper->setMaxIKSolutions(16);
      wrapper->setMinSolutionDistance(1);
      wrapper->setIKFrame(grasp_frame_transform, hand_frame);
      wrapper->properties().configureInitFrom(mtc::Stage::PARENT, { "eef", "group" });
      wrapper->properties().configureInitFrom(mtc::Stage::INTERFACE, { "target_pose" });
      grasp->insert(std::move(wrapper));
    }

    {
      // clang-format off
      auto stage =
          std::make_unique<mtc::stages::ModifyPlanningScene>("allow collision (hand,object)");
      stage->allowCollisions("object",
                             task.getRobotModel()
                                 ->getJointModelGroup(hand_group_name)
                                 ->getLinkModelNamesWithCollisionGeometry(),
                             true);
      // clang-format on
      grasp->insert(std::move(stage));
    }

    {
      auto stage = std::make_unique<mtc::stages::MoveTo>("close hand", interpolation_planner);
      stage->setGroup(hand_group_name);
      stage->setGoal("close");
      grasp->insert(std::move(stage));
    }

    {
      auto stage = std::make_unique<mtc::stages::ModifyPlanningScene>("attach object");
      stage->attachObject("object", hand_frame);
      attach_object_stage = stage.get();
      grasp->insert(std::move(stage));
    }

    {
      // clang-format off
      auto stage =
          std::make_unique<mtc::stages::MoveRelative>("lift object", cartesian_planner);
      // clang-format on
      stage->properties().configureInitFrom(mtc::Stage::PARENT, { "group" });
      stage->setMinMaxDistance(0.01, 0.1);
      stage->setIKFrame(hand_frame);
      stage->properties().set("marker_ns", "lift_object");

      // Set upward direction
      geometry_msgs::msg::Vector3Stamped vec;
      vec.header.frame_id = "world";
      vec.vector.z = 1.0;
      stage->setDirection(vec);
      grasp->insert(std::move(stage));
    }
    task.add(std::move(grasp));
  }

  {
    // clang-format off
    auto stage_move_to_place = std::make_unique<mtc::stages::Connect>(
        "move to place",
        mtc::stages::Connect::GroupPlannerVector{ { arm_group_name, sampling_planner },
                                                  { hand_group_name, sampling_planner } });
    // clang-format on
    stage_move_to_place->setTimeout(5.0);
    stage_move_to_place->properties().configureInitFrom(mtc::Stage::PARENT);
    task.add(std::move(stage_move_to_place));
  }

  {
    auto place = std::make_unique<mtc::SerialContainer>("place object");
    task.properties().exposeTo(place->properties(), { "eef", "group", "ik_frame" });
    // clang-format off
    place->properties().configureInitFrom(mtc::Stage::PARENT,
                                          { "eef", "group", "ik_frame" });
    // clang-format on

    /****************************************************
  ---- *               Generate Place Pose                *
     ***************************************************/
    {
      // Sample place pose
      auto stage = std::make_unique<mtc::stages::GeneratePlacePose>("generate place pose");
      stage->properties().configureInitFrom(mtc::Stage::PARENT);
      stage->properties().set("marker_ns", "place_pose");
      stage->setObject("object");

      geometry_msgs::msg::PoseStamped target_pose_msg;
      target_pose_msg.header.frame_id = "world";
      target_pose_msg.pose.position.x = -0.25;
      target_pose_msg.pose.position.y = 0.21;
      target_pose_msg.pose.orientation.w = 1.0;
      stage->setPose(target_pose_msg);
      stage->setMonitoredStage(attach_object_stage);  // Hook into attach_object_stage

      // Compute IK
      // clang-format off
      auto wrapper =
          std::make_unique<mtc::stages::ComputeIK>("place pose IK", std::move(stage));
      // clang-format on
      wrapper->setMaxIKSolutions(2);
      wrapper->setMinSolutionDistance(1.0);
      wrapper->setIKFrame("object");
      wrapper->properties().configureInitFrom(mtc::Stage::PARENT, { "eef", "group" });
      wrapper->properties().configureInitFrom(mtc::Stage::INTERFACE, { "target_pose" });
      place->insert(std::move(wrapper));
    }

    {
      auto stage = std::make_unique<mtc::stages::MoveTo>("open hand", interpolation_planner);
      stage->setGroup(hand_group_name);
      stage->setGoal("open");
      place->insert(std::move(stage));
    }

    {
      // clang-format off
      auto stage =
          std::make_unique<mtc::stages::ModifyPlanningScene>("forbid collision (hand,object)");
      stage->allowCollisions("object",
                             task.getRobotModel()
                                 ->getJointModelGroup(hand_group_name)
                                 ->getLinkModelNamesWithCollisionGeometry(),
                             false);
      // clang-format on
      place->insert(std::move(stage));
    }

    {
      auto stage = std::make_unique<mtc::stages::ModifyPlanningScene>("detach object");
      stage->detachObject("object", hand_frame);
      place->insert(std::move(stage));
    }

    {
      auto stage = std::make_unique<mtc::stages::MoveRelative>("retreat", cartesian_planner);
      stage->properties().configureInitFrom(mtc::Stage::PARENT, { "group" });
      stage->setMinMaxDistance(0.01, 0.1);
      stage->setIKFrame(hand_frame);
      stage->properties().set("marker_ns", "retreat");

      // Set retreat direction
      geometry_msgs::msg::Vector3Stamped vec;
      vec.header.frame_id = "world";
      vec.vector.z = 0.5;
      stage->setDirection(vec);
      place->insert(std::move(stage));
    }
    task.add(std::move(place));
  }

  {
    auto stage = std::make_unique<mtc::stages::MoveTo>("return home", interpolation_planner);
    stage->properties().configureInitFrom(mtc::Stage::PARENT, { "group" });
    stage->setGoal("home");
    task.add(std::move(stage));
  }
  return task;
}

int main(int argc, char** argv)
{
  rclcpp::init(argc, argv);

  rclcpp::NodeOptions options;
  options.automatically_declare_parameters_from_overrides(true);

  auto mtc_task_node = std::make_shared<MTCTaskNode>(options);
  rclcpp::executors::MultiThreadedExecutor executor;

  auto spin_thread = std::make_unique<std::thread>([&executor, &mtc_task_node]() {
    executor.add_node(mtc_task_node->getNodeBaseInterface());
    executor.spin();
    executor.remove_node(mtc_task_node->getNodeBaseInterface());
  });

  mtc_task_node->setupPlanningScene();
  // mtc_task_node->doTask();
  // 这里是多次执行任务的部分
  for (int i = 0; i < 10; ++i) {  // 执行10次抓取任务
    std::cout << "Executing task " << (i + 1) << "/10" << std::endl;
    mtc_task_node->doTask();  // 每次执行任务
    // 可以在这里检查任务是否成功
  }
  spin_thread->join();
  rclcpp::shutdown();
  return 0;
}
 
```

### 1.**头文件包含**：
```
#include <rclcpp/rclcpp.hpp>
#include <moveit/planning_scene/planning_scene.h>
#include <moveit/planning_scene_interface/planning_scene_interface.h>
#include <moveit/task_constructor/task.h>
#include <moveit/task_constructor/solvers.h>
#include <moveit/task_constructor/stages.h>
// ... (几何变换相关头文件)

- ROS 2核心 (`rclcpp`)
- MoveIt核心组件 (规划场景/任务构造器)
- MTC关键模块 (任务/求解器/阶段)
- 几何变换支持 (TF2/Eigen)
```
### 2.MTCTaskNode 类定义
#### **类结构设计**：
```
class MTCTaskNode {
public:
  MTCTaskNode(const rclcpp::NodeOptions& options);
  rclcpp::node_interfaces::NodeBaseInterface::SharedPtr getNodeBaseInterface();
  void doTask();
  void setupPlanningScene();
private:
  mtc::Task createTask();  // 核心任务构建函数
  mtc::Task task_;
  rclcpp::Node::SharedPtr node_;
};

- **节点封装**：将MTC任务封装在自定义类中
- **接口分离**：
    - `setupPlanningScene()`：设置碰撞环境
    - `createTask()`：构建任务管线
    - `doTask()`：执行任务流程
```

### 3. 成员函数实现
#### 1.构造函数
```
MTCTaskNode::MTCTaskNode(const rclcpp::NodeOptions& options)
  : node_{std::make_shared<rclcpp::Node>("mtc_node", options)} 
{}
- 创建ROS节点`mtc_node`
- 支持参数覆盖 (options)
```

#### 2. 规划场景设置
```
void MTCTaskNode::setupPlanningScene() {
  moveit_msgs::msg::CollisionObject object;
  object.id = "object";
  // ... 设置圆柱体参数 (半径0.01m, 高0.04m)
  object.primitives[0].type = shape_msgs::msg::SolidPrimitive::CYLINDER;
  object.primitives[0].dimensions = {0.04, 0.01}; 
  
  geometry_msgs::msg::Pose pose;
  pose.position.x = 0.15;  // 世界坐标系中的位置
  pose.position.y = 0.25;
  
  moveit::planning_interface::PlanningSceneInterface psi;
  psi.applyCollisionObject(object);
}

- **创建被抓取物体**：
    - ID: "object"
    - 类型: 圆柱体
    - 位姿: (0.15, 0.25, 0) 世界坐标系
    - 尺寸: 直径0.02m × 高0.04m
```

#### 3. 任务执行流程
```
void MTCTaskNode::doTask() {
  task_ = createTask();  // 构建任务
  
  // 1. 初始化
  task_.init(); 
  
  // 2. 规划 (最多5个解)
  if (!task_.plan(5)) {
    RCLCPP_ERROR(LOGGER, "Planning failed");
    return;
  }
  
  // 3. 可视化最优解
  task_.introspection().publishSolution(*task_.solutions().front());
  
  // 4. 执行
  auto result = task_.execute(*task_.solutions().front());
  if (result.val != moveit_msgs::msg::MoveItErrorCodes::SUCCESS) {
    RCLCPP_ERROR(LOGGER, "Execution failed");
  }
}

- **四阶段工作流**：初始化→规划→可视化→执行
- **容错机制**：各阶段错误检测与日志记录
```

### 4.核心：任务构建 (`createTask()`)
#### 1.任务基础配置
```
mtc::Task task;
task.loadRobotModel(node_);  // 加载机器人模型

// 组定义
const auto& arm_group_name = "dummy2_arm";
const auto& hand_group_name = "hand";
const auto& hand_frame = "grasp_frame";  // 夹爪坐标系

// 全局属性
task.setProperty("group", arm_group_name);
task.setProperty("eef", hand_group_name);
task.setProperty("ik_frame", hand_frame);  // IK参考坐标系
```

#### 2.求解器配置
```
// 三种运动规划器
auto sampling_planner = std::make_shared<mtc::solvers::PipelinePlanner>(node_);  // 采样规划
auto interpolation_planner = std::make_shared<mtc::solvers::JointInterpolationPlanner>();  // 关节插值

auto cartesian_planner = std::make_shared<mtc::solvers::CartesianPath>();  // 笛卡尔规划
cartesian_planner->setMaxVelocityScalingFactor(1.0);  // 速度比例因子
cartesian_planner->setStepSize(.01);  // 路径分辨率
```

### 5.任务阶段分解
#### 1. 初始状态
```
auto stage_state_current = std::make_unique<mtc::stages::CurrentState>("current");
task.add(std::move(stage_state_current));
- 获取当前关节状态
```

#### 2. 打开夹爪
```
auto stage_open_hand = std::make_unique<mtc::stages::MoveTo>("open hand", interpolation_planner);
stage_open_hand->setGroup(hand_group_name);
stage_open_hand->setGoal("open");  // 预设姿态
task.add(std::move(stage_open_hand));
```

#### 3. 向抓取点移动
```
auto stage_move_to_pick = std::make_unique<mtc::stages::Connect>("move to pick",...);
stage_move_to_pick->setTimeout(5.0);  // 5秒超时
task.add(std::move(stage_move_to_pick));
```

#### 4. 抓取序列容器 (核心)
```
auto grasp = std::make_unique<mtc::SerialContainer>("pick object");
{
  // 4.1 接近物体 (笛卡尔移动)
  auto stage_approach = std::make_unique<mtc::stages::MoveRelative>("approach object", cartesian_planner);
  geometry_msgs::msg::Vector3Stamped vec;
  vec.vector.y = 1.0;  // Y轴方向 (夹爪坐标系)
  stage_approach->setDirection(vec);
  
  // 4.2 生成抓取位姿
  auto stage_grasp_pose = std::make_unique<mtc::stages::GenerateGraspPose>("generate grasp pose");
  stage_grasp_pose->setObject("object");  // 目标物体
  
  // 4.3 抓取位姿IK解算
  auto wrapper = std::make_unique<mtc::stages::ComputeIK>("grasp pose IK", std::move(stage_grasp_pose));
  Eigen::Isometry3d grasp_transform;  // 物体→夹爪的变换
  grasp_transform.translation().y() = 0.03;  // 夹爪在物体坐标系中的偏移
  
  // 4.4 允许碰撞 (抓取时)
  auto stage_allow_collision = std::make_unique<mtc::stages::ModifyPlanningScene>("allow collision");
  
  // 4.5 闭合夹爪
  auto stage_close_hand = std::make_unique<mtc::stages::MoveTo>("close hand");
  stage_close_hand->setGoal("close");
  
  // 4.6 附着物体
  auto stage_attach = std::make_unique<mtc::stages::ModifyPlanningScene>("attach object");
  stage_attach->attachObject("object", hand_frame);
  
  // 4.7 抬升物体
  auto stage_lift = std::make_unique<mtc::stages::MoveRelative>("lift object");
  geometry_msgs::msg::Vector3Stamped lift_vec;
  lift_vec.vector.z = 1.0;  // Z轴方向 (世界坐标系)
}
task.add(std::move(grasp));
```

#### 5. 放置序列容器
```
auto place = std::make_unique<mtc::SerialContainer>("place object");
{
  // 5.1 生成放置位姿
  auto stage_place_pose = std::make_unique<mtc::stages::GeneratePlacePose>("generate place pose");
  geometry_msgs::msg::PoseStamped target_pose;
  target_pose.pose.position.x = -0.25;  // 放置目标位置
  target_pose.pose.position.y = 0.21;
  
  // 5.2 放置位姿IK解算
  // ...
  
  // 5.3 打开夹爪
  auto stage_open_hand = std::make_unique<mtc::stages::MoveTo>("open hand");
  
  // 5.4 禁止碰撞
  auto stage_forbid_collision = std::make_unique<mtc::stages::ModifyPlanningScene>("forbid collision");
  
  // 5.5 分离物体
  auto stage_detach = std::make_unique<mtc::stages::ModifyPlanningScene>("detach object");
  
  // 5.6 撤离
  auto stage_retreat = std::make_unique<mtc::stages::MoveRelative>("retreat");
  geometry_msgs::msg::Vector3Stamped retreat_vec;
  retreat_vec.vector.z = 0.5;  // 向上撤离
}
```

#### 6. 返回初始位置
```
auto stage_home = std::make_unique<mtc::stages::MoveTo>("return home");
stage_home->setGoal("home");  // 预设的"home"位形
task.add(std::move(stage_home));
```

### 6.主函数
```
int main(int argc, char** argv) {
  rclcpp::init(argc, argv);
  
  // 1. 创建节点
  auto mtc_task_node = std::make_shared<MTCTaskNode>(options);
  
  // 2. 多线程执行器
  rclcpp::executors::MultiThreadedExecutor executor;
  
  // 3. 专用线程处理回调
  auto spin_thread = std::make_unique<std::thread>([&](){
    executor.add_node(mtc_task_node->getNodeBaseInterface());
    executor.spin();
  });
  
  // 4. 设置场景并循环执行任务
  mtc_task_node->setupPlanningScene();
  for (int i = 0; i < 10; ++i) {  // 执行10次
    mtc_task_node->doTask();
  }
  
  // 5. 清理
  spin_thread->join();
  rclcpp::shutdown();
}
```

### MTC阶段话设计
![[Pasted image 20250804012228.png]]