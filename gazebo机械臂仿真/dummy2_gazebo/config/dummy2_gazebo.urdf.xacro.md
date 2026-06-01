```
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="dummy2-gripperv2">
    <xacro:arg name="initial_positions_file" default="/home/jasonlee/Moveit2YoloObb/ros2_project/src/dummy2-gripperv2_moveit_config/config/initial_positions.yaml"/>
    <xacro:property name="PI" value="3.14159274"/>
    <!-- Import panda urdf file -->
    <xacro:include filename="$(find dummy2-gripperv2_description)/urdf/dummy2-gripperv2.xacro"/>
    <!-- Lidar -->
    <!-- <xacro:include filename="$(find robot_description)/urdf/lidar/lidar.xacro"/>
    <xacro:lidar_v0 parent="panda_link7">
        <origin xyz="-0.2 -0.2 0.2" rpy="0 ${PI/2} 0"/>
    </xacro:lidar_v0> -->
    <xacro:include filename="$(find dummy2-gripperv2_description)/urdf/camera/camera.xacro"/>
    <!-- Camera -->
    <xacro:camera_v0 parent="base_link">
        <origin xyz="0.2 0.6 0.7" rpy="0 ${PI/2} 0"/>
    </xacro:camera_v0>
    <xacro:camera_gazebo_v0/>
    <gazebo>
        <plugin filename="gz_ros2_control-system" name="gz_ros2_control::GazeboSimROS2ControlPlugin">
            <parameters>$(find dummy2-gripperv2_moveit_config)/config/ros2_controllers.yaml</parameters>
        </plugin>
        <plugin
        filename="gz-sim-joint-state-publisher-system"
        name="gz::sim::systems::JointStatePublisher">
    </plugin>
        <plugin
        filename="gz-sim-pose-publisher-system"
        name="gz::sim::systems::PosePublisher">
            <publish_link_pose>true</publish_link_pose>
            <use_pose_vector_msg>true</use_pose_vector_msg>
            <publish_nested_model_pose>true</publish_nested_model_pose>
        </plugin>
        <plugin filename="ignition-gazebo-sensors-system" name="ignition::gazebo::systems::Sensors">
            <render_engine>ogre2</render_engine>
        </plugin>
    </gazebo>
    <gazebo reference="finger1">
        <mu1>100.0</mu1>
        <mu2>100.0</mu2>
        <kp>30000</kp>
        <kd>15</kd>
    </gazebo>
    <gazebo reference="finger2">
        <mu1>100.0</mu1>
        <mu2>100.0</mu2>
        <kp>30000</kp>
        <kd>15</kd>
    </gazebo>
    <xacro:include filename="dummy2_gazebo.friction.xacro" />
    <xacro:dummy2-gripperv2_ros2_control name="GazeboSimSystem" initial_positions_file="$(arg initial_positions_file)"/>
</robot>
```