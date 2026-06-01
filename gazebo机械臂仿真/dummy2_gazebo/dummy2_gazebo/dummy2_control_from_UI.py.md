```
#!/usr/bin/env python3
import threading
import time
import rclpy
from rclpy.node import Node
from tf_transformations import quaternion_from_euler
from std_msgs.msg import Float64MultiArray
from geometry_msgs.msg import Point, Quaternion
from geometry_msgs.msg import Pose, PoseStamped
from scipy.spatial.transform import Rotation as R
from pymoveit2 import MoveIt2
from rclpy.callback_groups import ReentrantCallbackGroup
from std_msgs.msg import String


class Controller(Node):
    def __init__(self):
        super().__init__("yolo_pick_node")

        self.moveit_callback_group = ReentrantCallbackGroup()
        self.subscription = self.create_subscription(
            Float64MultiArray,
            "/target_point",
            self.listener_callback,
            10,
        )
        self.arm = MoveIt2(
            node=self,
            joint_names=["joint1", "joint2", "joint3", "joint4", "joint5", "joint6"],
            base_link_name="base_link",
            end_effector_name="grasp_frame",
            group_name="dummy2_arm",
            callback_group=self.moveit_callback_group,
            use_move_group_action=True,
        )

        self.hand = MoveIt2(
            node=self,
            # joint_names=hand_joint_names,
            joint_names=["figer1", "figer2"],
            base_link_name="base_link",
            end_effector_name="grasp_frame",
            group_name="hand",
            callback_group=self.moveit_callback_group, 
            use_move_group_action=True,
        )

        self.setup_params()
        # self.setup_poses()  

        self.arm.planner_id = "RRTConnect"
        self.arm.max_velocity = 0.5
        self.arm.max_acceleration = 0.5
        self.arm.allowed_planning_time = 15.0
        self.arm.position_tolerance = 2
        self.arm.orientation_tolerance = 2

        self.get_logger().info("✅ Controller initialized")

    def setup_params(self):
         # 参数配置
        self.height = 0.12
        self.pick_height = 0.03
        self.carrying_height = 0.12
        # self.init_angle = -0.3825
        self.init_angle = -1.5888
        self.home_joints = [0.0, 0.0, 0.0, 0.0, 0.0, 0.0]

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
    
    
    def pose_to_pose_stamped(self, pose):
        """将Pose转换为PoseStamped"""
        pose_stamped = PoseStamped()
        pose_stamped.header.frame_id = "base_link"
        pose_stamped.header.stamp = self.get_clock().now().to_msg()
        pose_stamped.pose = pose
        return pose_stamped
    

    def move_to(self, target_pose, cartesian=False, action_name="移动"):

        target_pose = self.pose_to_pose_stamped(target_pose)

        try:
            self.arm.move_to_pose(pose=target_pose, cartesian=cartesian)
            self.arm.wait_until_executed()
            self.get_logger().info(f'✓ {action_name}完成')
            time.sleep(3.0)
            return True

        except Exception as e:
            self.get_logger().error(f'✗ {action_name}失败: {e}')
            return False

    
    def gripper_action(self, open_gripper=True):
        """控制夹爪张开或闭合"""
        action = "张开夹爪" if open_gripper else "闭合夹爪"
        self.get_logger().info(f'正在执行: {action}')
        
        positions = [0.028, -0.028] if open_gripper else [0.0, 0.0]
        try:
            self.hand.move_to_configuration(positions)
            self.hand.wait_until_executed()
            self.get_logger().info(f'✓ {action}完成')
            time.sleep(3.0)
            return True
        except Exception as e:
            time.sleep(3.0)
            return True

    def move_to_home(self):
        """返回 home 位置"""
        self.get_logger().info("正在返回 home 位置")
        self.arm.move_to_configuration(self.home_joints)  # Move to home joints
        self.arm.wait_until_executed()
        time.sleep(3.0)


    def listener_callback(self, data):
        try:
            x, y, yaw = data.data[0], data.data[1], data.data[2]
            self.get_logger().info(f"🎯 New target: x={x:.3f}, y={y:.3f}, yaw={yaw:.3f}")

            grasping_roll,grasping_pitch,grasping_yaw,= -90,0.0,(yaw + self.init_angle)*180/3.1415
            # grasping_roll,grasping_pitch,grasping_yaw,= -90,0.0,-90.0

            # 定义所有关键姿态
            self.poses = {
                'height': self.make_pose(x, y, self.height, grasping_roll, grasping_pitch, grasping_yaw),
                'pick_height': self.make_pose(x, y, self.pick_height, grasping_roll, grasping_pitch, grasping_yaw),
                'carrying_height': self.make_pose(x, y, self.carrying_height, grasping_roll, grasping_pitch, grasping_yaw),
                'place': self.make_pose(0.1, -0.1, self.carrying_height, grasping_roll, grasping_pitch, grasping_yaw)
            }
            # 执行抓取流程
            # self.control_loop()

            # # 执行抓取序列
            self.gripper_action(True) 
            self.move_to(self.poses['height'],cartesian=False,action_name='height') 
            self.move_to(self.poses['pick_height'],cartesian=True,action_name='pick_height') 
            self.gripper_action(False) 
            self.move_to(self.poses['carrying_height'],cartesian=True,action_name='carrying_height') 
            self.move_to(self.poses['place'],cartesian=False,action_name='place') 
            self.gripper_action(True)
            self.move_to_home()
        finally:
            self.is_executing = False


def main():
    rclpy.init(args=None)
    controller = Controller()
    
    executor = rclpy.executors.MultiThreadedExecutor()
    executor.add_node(controller)
    
    try:
        executor.spin()  # ← 直接spin，不用单独线程
    except KeyboardInterrupt:
        pass
    finally:
        rclpy.shutdown()


if __name__ == "__main__":
    main()




```