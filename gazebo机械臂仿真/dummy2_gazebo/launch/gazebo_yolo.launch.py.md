```

from moveit_configs_utils import MoveItConfigsBuilder
from ament_index_python.packages import get_package_share_directory
from launch_ros. actions import Node
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.actions import RegisterEventHandler, SetEnvironmentVariable
import os
from launch.actions import DeclareLaunchArgument, ExecuteProcess, IncludeLaunchDescription
from pathlib import Path
from launch.substitutions import LaunchConfiguration

def generate_launch_description():
    packagepath = get_package_share_directory('dummy2_gazebo')
    thispackagepath = get_package_share_directory('dummy2-gripperv2_moveit_config')
    print(packagepath)

    # 找到robot_description功能包
    robot_description_path = os.path.join(
        get_package_share_directory('dummy2-gripperv2_description'))
    
    # 找到dummy2_gazebo功能包
    arm_robot_sim_path = os.path.join(
        get_package_share_directory('dummy2_gazebo'))
    
    arguments = LaunchDescription([
                DeclareLaunchArgument('world', default_value='arm_on_the_table',
                          description='Gz sim World'),
           ]
    )
    

    # Set gazebo sim resource path
    gazebo_resource_path = SetEnvironmentVariable(
        name='GZ_SIM_RESOURCE_PATH',
        value=[
            os.path.join(arm_robot_sim_path, 'worlds'), ':' +
            str(Path(robot_description_path).parent.resolve())
            ]
        )
    # Load the robot configuration
    moveit_config =(MoveItConfigsBuilder("dummy2-gripperv2", package_name="dummy2-gripperv2_moveit_config")
                .robot_description(packagepath + '/config/dummy2_gazebo.urdf.xacro')
                .moveit_cpp(arm_robot_sim_path + "/config/controller_setting.yaml") # moveit settings
                .robot_description_semantic('config/dummy2-gripperv2.srdf').to_moveit_configs()
    )

    #启动Gazebo
    gazebo_node = IncludeLaunchDescription(
                PythonLaunchDescriptionSource([os.path.join(get_package_share_directory('ros_gz_sim'), 'launch'), '/gz_sim.launch.py']),
                launch_arguments=[
                    ('gz_args', [LaunchConfiguration('world'),
                                 '.sdf',
                                #  ' -v 4',
                                 ' -r',]
                    )
                ]
             )


    moveit_py_node = Node(
        name="moveit_py",
        package="dummy2_gazebo",
        executable="dummy2_control_from_UI",
        output="screen",
        parameters=[moveit_config.to_dict(),
                    # {"use_sim_time": True},
                    ],
    )
    #将机械臂添加到Gazebo
    robot_to_gazebo_node = Node(
        package='ros_gz_sim',
        executable='create',
        output='both',
        arguments=['-string', moveit_config.robot_description["robot_description"],
                   '-x', '0.05',
                   '-y', '0.35',
                   '-z', '1.02',
                   '-R', '0.0',
                   '-P', '0.0',
                   '-Y', '0.0',
                   '-name', 'dummy2_arm',
                   '-allow_renaming', 'false'],
    )

    # CLock Bridge
    clock_bridge_node = Node(
        package='ros_gz_bridge',
        executable='parameter_bridge',
        arguments=['/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock'],
        output='both',
        # parameters=[{'use_sim_time': True}],
    )

    # image Bridge
    image_bridge_node = Node(
        package='ros_gz_bridge',
        executable='parameter_bridge',
        arguments=['/image_raw@sensor_msgs/msg/Image@gz.msgs.Image'], 
        output='screen'
    )

    #发布机械臂状态
    robot_desc_node = Node(
        package="robot_state_publisher",
        executable="robot_state_publisher",
        name="robot_state_publisher",
        output="both",
        parameters=[moveit_config.robot_description,
                    {'use_sim_time': True},  # 必须使用仿真时间
                    { "publish_frequency":30.0,},
                    ],
    )

    # Launch RViz
    rviz_node = Node(
        package="rviz2",
        executable="rviz2",
        output="log",
        arguments=["-d", packagepath + '/rviz/dummy2_gazebo.rviz'],
        parameters=[
            moveit_config.robot_description,
            moveit_config.robot_description_semantic,
            moveit_config.robot_description_kinematics,
            moveit_config.planning_pipelines,
            moveit_config.joint_limits,
            {'use_sim_time': True},
        ],
    )

    #ros2_ controller manger 节点
    ros2_control_node = Node(
        package= "controller_manager",
        executable="ros2_control_node",
        parameters=[thispackagepath+ '/config/ros2_controllers.yaml',
                    {'use_sim_time': True}],
        output= "both",
    )


    #启动关节状态发布器，arm组控制器，夹抓控制器
    controller_spawner_node = Node(
        package ="controller_manager",
        executable= "spawner",
        arguments= [
           "joint_state_broadcaster","dummy2_arm_controller","hand_controller"
        ],
        parameters=[{'use_sim_time': True}],
        output="screen",
    )

    load_controllers = []
    for controller in [
        "joint_state_broadcaster","dummy2_arm_controller","hand_controller"
    ]:
        load_controllers += [
            ExecuteProcess(
                cmd=["ros2 run controller_manager spawner {}".format(controller)],
                shell=True,
                output="screen",
            )
        ]

    #启动move_ group node/action server
    move_group_node = Node(
        package="moveit_ros_move_group",
        executable="move_group",
        output="screen",
        parameters=[moveit_config.to_dict(),
                    {'use_sim_time': True}],
    )

    # # 静态tf转换：world到base_link
    static_tf = Node( 
        package="tf2_ros",
        executable="static_transform_publisher",
        name="static_transform_publisher",
        output="log",
        arguments=["--frame-id", "world", "--child-frame-id", "base_link"],
        parameters=[{"use_sim_time": True},],
    )

    
    return LaunchDescription([
        gazebo_resource_path,
        arguments,
        gazebo_node,
        robot_to_gazebo_node,
        clock_bridge_node,
        robot_desc_node,
        rviz_node,
        ros2_control_node,
        # controller_spawner_node,
        move_group_node,
        image_bridge_node,
        # moveit_py_node,
        # static_tf
        ]
        + load_controllers
    )








```