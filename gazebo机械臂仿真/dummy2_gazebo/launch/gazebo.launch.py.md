```
# gazebo.launch.py
import os
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription, TimerAction
from launch.launch_description_sources import PythonLaunchDescriptionSource
from ament_index_python.packages import get_package_share_directory
from moveit_configs_utils import MoveItConfigsBuilder

def generate_launch_description():
    # 获取当前包路径
    this_package_path = get_package_share_directory('dummy2_gazebo')  
    robot_desc_path=get_package_share_directory('dummy2-gripperv2_description')

    # 启动Gazebo
    gazebo_node = IncludeLaunchDescription(
        PythonLaunchDescriptionSource([
            get_package_share_directory('ros_gz_sim') + '/launch/gz_sim.launch.py']),
        launch_arguments=[('gz_args', 'empty.sdf -r')]  # 使用自定义空世界
    )

    clock_bridge_node = Node(
    package='ros_gz_bridge',
    executable='parameter_bridge',
    arguments=['/world/empty/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock'],
    output='both',
    parameters=[{'use_sim_time': True}],
    remappings=[('/world/empty/clock', '/clock')]  
    )

    packagepath = get_package_share_directory('dummy2-gripperv2_moveit_config')  

    # MoveIt配置
    moveit_config = MoveItConfigsBuilder("dummy2-gripperv2", package_name="dummy2-gripperv2_moveit_config") \
                    .robot_description(this_package_path + '/config/dummy2_gazebo.friction.urdf.xacro') \
                    .robot_description_semantic('config/dummy2-gripperv2.srdf') \
                    .moveit_cpp(this_package_path + "/config/movelt_cpp.yaml") \
                    .to_moveit_configs()

    #　将机械臂添加到Gazebo
    gz_urdf= moveit_config.robot_description['robot_description'].replace('package://dummy2-gripperv2_description',robot_desc_path) 
    robot_to_gazebo_node = Node(
            package='ros_gz_sim',
            executable='create',
            arguments=['-string', gz_urdf,
               '-x','0.0','-y','0.0','-z','0.0',
               '-R','0','-P','0','-Y','0',   # yaw=90°
               '-name','dummy2_arm']
    )
    # 发布机械臂状态
    robot_desc_node = Node(
        package="robot_state_publisher",
        executable="robot_state_publisher",
        name="robot_state_publisher",
        output="both",
        parameters=[moveit_config.robot_description,
                     {'use_sim_time': True},    #必须使用仿真时间
                     { "publish_frequency":100.0,},
                     ],
    )

    # 启动RViz
    rviz_node = Node(
        package="rviz2",
        executable="rviz2",
        output="log",
        arguments=["-d", this_package_path + '/rviz/pick_drop_moveit.rviz'],
        parameters=[
            moveit_config.robot_description,
            moveit_config.robot_description_semantic,
            moveit_config.robot_description_kinematics,
            moveit_config.planning_pipelines,
            moveit_config.joint_limits,
            {'use_sim_time': True},
        ],
    )

    #启动关节状态发布器，arm组控制器，夹抓控制器
    controller_spawner_node = Node(
        package="controller_manager",
        executable="spawner",
        arguments=[
            "joint_state_broadcaster","dummy2_arm_controller","hand_controller"
        ],
        parameters=[{'use_sim_time': True}],
        output="screen",
    )

    # 启动move_group
    move_group_node = Node(
        package="moveit_ros_move_group",
        executable="move_group",
        output="screen",
        parameters=[moveit_config.to_dict(), {'use_sim_time': True}],
    )

    return LaunchDescription([
        gazebo_node,  # 启动Gazebo仿真环境
        clock_bridge_node,  # 时钟桥接
        robot_to_gazebo_node,#启动gazebo环境机械臂
        robot_desc_node, #启动机械臂状态节点
        rviz_node,  # 启动RViz
        controller_spawner_node,#启动关节状态发布器
        move_group_node,  # 启动MoveIt的move_group
    ])

```