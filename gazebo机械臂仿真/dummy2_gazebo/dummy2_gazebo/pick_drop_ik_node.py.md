```
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from rclpy.callback_groups import ReentrantCallbackGroup
from rclpy.executors import MultiThreadedExecutor
from geometry_msgs.msg import Pose, PoseStamped
from std_msgs.msg import String
from pymoveit2 import MoveIt2
from scipy.spatial.transform import Rotation as R
import time
from enum import Enum


class TaskState(Enum):
    IDLE = "idle"
    MOVING_TO_BOX_ABOVE = "moving_to_box_above"
    MOVING_TO_BOX = "moving_to_box"
    GRASPING = "grasping"
    LIFTING_BOX = "lifting_box"
    MOVING_TO_INTERMEDIATE = "moving_to_intermediate" 
    MOVING_TO_CASE = "moving_to_case"
    RELEASING = "releasing"
    RETURNING_HOME = "returning_home"
    COMPLETED = "completed"
    ERROR = "error"


class PenBoxGraspingNode(Node):
    def __init__(self):
        super().__init__('pick_drop_ik')
        
        self.callback_group = ReentrantCallbackGroup()
        time.sleep(2.0)
        
        self.setup_moveit()
        self.setup_params()
        self.setup_poses()  
        
        self.current_state = TaskState.IDLE
        self.state_publisher = self.create_publisher(String, '/task_state', 100)
        self.create_timer(4.0, self.control_loop)
        
        self.get_logger().info('抓取节点启动完成')

    def setup_moveit(self):
        """初始化MoveIt接口"""
        try:
            self.moveit2_arm = MoveIt2(
                node=self, 
                joint_names=["joint1", "joint2", "joint3", "joint4", "joint5", "joint6"],
                base_link_name="base_link", 
                end_effector_name="grasp_frame",
                group_name="dummy2_arm", 
                callback_group=self.callback_group,
                use_move_group_action=True
            )
            
            self.moveit2_arm.planner_id = "RRTConnect"
            self.moveit2_arm.max_velocity = 0.5
            self.moveit2_arm.max_acceleration = 0.5
            self.moveit2_arm.allowed_planning_time = 15.0
            self.moveit2_arm.position_tolerance = 2
            self.moveit2_arm.orientation_tolerance = 2
            
            self.moveit2_gripper = MoveIt2(
                node=self, 
                joint_names=["figer1", "figer2"],
                base_link_name="base_link", 
                end_effector_name="grasp_frame",
                group_name="hand", 
                callback_group=self.callback_group,
                use_move_group_action=True
            )
            
            self.get_logger().info('MoveIt接口初始化成功')
        except Exception as e:
            self.get_logger().error(f'MoveIt初始化失败: {e}')
            import traceback
            self.get_logger().error(traceback.format_exc())

    def setup_params(self):
        """设置基础参数"""
        # 目标位置
        self.BOX_POS = (0.2, 0.0, 0.04)
        self.CASE_POS = (0.0, 0.50, 0.08)
        
        # 运动参数
        self.grasp_offset = 0.007
        self.safe_height = 0.1
        
        self.home_joints = [0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
        self.action_delay = 2.0

        self.INTERMEDIATE_POS = (0.2, 0.25, 0.16)  # 自定义中间位置坐标
      
        

    def make_pose(self, x, y, z, roll, pitch, yaw):
        p = Pose()
        p.position.x = float(x)
        p.position.y = float(y)
        p.position.z = float(z)
        
        # 欧拉角转四元数
        r = R.from_euler('xyz', [roll, pitch, yaw], degrees=True)
        quat = r.as_quat()
        p.orientation.x = float(quat[0])
        p.orientation.y = float(quat[1])
        p.orientation.z = float(quat[2])
        p.orientation.w = float(quat[3])
        
        return p

    def setup_poses(self):
        """集中定义所有任务姿态"""
        # 抓取box的姿态（垂直向下）
        box_roll, box_pitch, box_yaw = -90, 0, 0
        
        # 中间位置姿态（用于前往中间点）
        intermediate_roll, intermediate_pitch, intermediate_yaw = 0, 0, 0

        # 放置case的姿态（标准姿态）
        case_roll, case_pitch, case_yaw = 0, 0, 0
        
        
        
        # 定义所有关键姿态
        self.poses = {

            # 抓取阶段
            'box_above': self.make_pose(
                self.BOX_POS[0],
                self.BOX_POS[1],
                self.BOX_POS[2] + self.safe_height,
                box_roll, box_pitch, box_yaw
            ),

            'box_grasp': self.make_pose(
                self.BOX_POS[0],
                self.BOX_POS[1],
                self.BOX_POS[2] + self.grasp_offset,
                box_roll, box_pitch, box_yaw
            ),
            
            'box_lift': self.make_pose(
                self.BOX_POS[0],
                self.BOX_POS[1],
                self.BOX_POS[2] + self.safe_height,
                box_roll, box_pitch, box_yaw
            ),
            
            # 中间位置阶段
            'intermediate_place': self.make_pose(
                self.INTERMEDIATE_POS[0],
                self.INTERMEDIATE_POS[1],
                self.INTERMEDIATE_POS[2],
                intermediate_roll, intermediate_pitch, intermediate_yaw
            ),

            # case放置阶段
            'case_place': self.make_pose(
                self.CASE_POS[0],
                self.CASE_POS[1],
                self.CASE_POS[2] + self.safe_height,
                case_roll, case_pitch, case_yaw
            ),
        }
        

    def pose_to_pose_stamped(self, pose):
        """将Pose转换为PoseStamped"""
        pose_stamped = PoseStamped()
        pose_stamped.header.frame_id = "base_link"
        pose_stamped.header.stamp = self.get_clock().now().to_msg()
        pose_stamped.pose = pose
        return pose_stamped

    def move_to_pose(self, target_pose, cartesian=False, action_name="移动"):

        target_pose = self.pose_to_pose_stamped(target_pose)

        try:
            self.moveit2_arm.move_to_pose(pose=target_pose, cartesian=cartesian)
            # self.moveit2_arm.wait_until_executed()
            self.get_logger().info(f'✓ {action_name}完成')
            time.sleep(self.action_delay)
            return True

        except Exception as e:
            self.get_logger().error(f'✗ {action_name}失败: {e}')
            return False

    def control_gripper(self, open_gripper=True):
        """控制夹爪张开或闭合"""
        action = "张开夹爪" if open_gripper else "闭合夹爪"
        self.get_logger().info(f'正在执行: {action}')
        
        positions = [0.028, -0.028] if open_gripper else [0.0, 0.0]
        try:
            self.moveit2_gripper.move_to_configuration(positions)
            # self.moveit2_gripper.wait_until_executed()
            self.get_logger().info(f'✓ {action}完成')
            time.sleep(self.action_delay)
            return True
        except Exception as e:
            time.sleep(self.action_delay)
            return True

    def go_home(self):
        """返回home位置"""
        self.get_logger().info('正在执行: 返回HOME')
        try:
            self.moveit2_arm.move_to_configuration(self.home_joints)
            # self.moveit2_arm.wait_until_executed()
            self.get_logger().info('✓ 返回HOME完成')
            time.sleep(self.action_delay)
            return True
        except Exception as e:
            self.get_logger().error(f'返回HOME失败: {e}')
            return False

    def control_loop(self):
        """主控制循环"""
        state_msg = String()
        state_msg.data = self.current_state.value
        self.state_publisher.publish(state_msg)
        
        try:
            if self.current_state == TaskState.IDLE:
                self.get_logger().info("=== 开始抓取任务 ===")
                self.control_gripper(True)
                self.current_state = TaskState.MOVING_TO_BOX_ABOVE
            
            # ===== 阶段1.5: 移动到box抓取位置 =====
            elif self.current_state == TaskState.MOVING_TO_BOX_ABOVE:
                if self.move_to_pose(
                    self.poses['box_above'], 
                    cartesian=False, 
                    action_name="移动到box_above抓取位置"
                ):
                    self.current_state = TaskState.MOVING_TO_BOX
                else:
                    self.current_state = TaskState.ERROR

            
            # ===== 阶段2.0: 移动到box抓取位置 =====
            elif self.current_state == TaskState.MOVING_TO_BOX:
                if self.move_to_pose(
                    self.poses['box_grasp'], 
                    cartesian=False, 
                    action_name="移动到box抓取位置"
                ):
                    self.current_state = TaskState.GRASPING
                else:
                    self.current_state = TaskState.ERROR
            
            # ===== 阶段2: 闭合夹爪 =====
            elif self.current_state == TaskState.GRASPING:
                self.control_gripper(False)
                self.current_state = TaskState.LIFTING_BOX
            
            # ===== 阶段3: 提升box =====
            elif self.current_state == TaskState.LIFTING_BOX:
                if self.move_to_pose(
                    self.poses['box_lift'], 
                    cartesian=False, 
                    action_name="提升box"
                ):
                    self.current_state = TaskState.MOVING_TO_CASE
                else:
                    self.current_state = TaskState.ERROR
            
            # ===== 阶段3.5: 移动到中间位置 =====
            elif self.current_state == TaskState.MOVING_TO_INTERMEDIATE:
                if self.move_to_pose(
                    self.poses['intermediate_place'], 
                    cartesian=False, 
                    action_name="移动到中间位置"
                ):
                    self.current_state = TaskState.MOVING_TO_CASE
                else:
                    self.current_state = TaskState.ERROR

            # ===== 阶段4: 移动到case放置位置 =====
            elif self.current_state == TaskState.MOVING_TO_CASE:
                if self.move_to_pose(
                    self.poses['case_place'], 
                    cartesian=False, 
                    action_name="移动到case放置位置"
                ):
                    self.current_state = TaskState.RELEASING
                else:
                    self.current_state = TaskState.ERROR
            
            # ===== 阶段5: 释放box =====
            elif self.current_state == TaskState.RELEASING:
                self.control_gripper(True)
                self.current_state = TaskState.RETURNING_HOME
            
            # ===== 阶段6: 返回home =====
            elif self.current_state == TaskState.RETURNING_HOME:
                if self.go_home():
                    self.current_state = TaskState.COMPLETED
                else:
                    self.current_state = TaskState.ERROR
            
            # ===== 任务完成 =====
            elif self.current_state == TaskState.COMPLETED:
                self.get_logger().info("=== 任务完成 ===")
                time.sleep(3.0)
                self.current_state = TaskState.IDLE
            
            # ===== 错误恢复 =====
            elif self.current_state == TaskState.ERROR:
                self.get_logger().error("!!! 任务出错，执行恢复 !!!")
                self.control_gripper(True)
                time.sleep(self.action_delay)
                if self.go_home():
                    time.sleep(3.0)
                    self.current_state = TaskState.IDLE
                else:
                    time.sleep(3.0)
                    
        except Exception as e:
            self.get_logger().error(f'控制循环异常: {e}')
            import traceback
            self.get_logger().error(f'详细错误:\n{traceback.format_exc()}')
            self.current_state = TaskState.ERROR


def main(args=None):
    rclpy.init(args=args)
    try:
        node = PenBoxGraspingNode()
        executor = MultiThreadedExecutor(num_threads=4)
        executor.add_node(node)
        node.get_logger().info("=== 抓取节点就绪 ===")
        executor.spin()
    except KeyboardInterrupt:
        node.get_logger().info("任务被中断")
    except Exception as e:
        print(f"程序异常: {e}")
        import traceback
        traceback.print_exc()
    finally:
        rclpy.shutdown()


if __name__ == "__main__":
    main()






```