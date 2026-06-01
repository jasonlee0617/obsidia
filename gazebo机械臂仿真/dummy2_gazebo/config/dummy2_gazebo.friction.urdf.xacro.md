```
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="dummy2-gripperv2">
    <xacro:arg name="initial_positions_file" default="/home/jasonlee/dummy2/ros2/dummy2_hand_eye_calibration_ws/src/dummy2-gripperv2_moveit_config/config/initial_positions.yaml" />

    <!-- Import dummy2-gripperv2 urdf file -->
    <xacro:include filename="$(find dummy2-gripperv2_description)/urdf/dummy2-gripperv2.xacro" />

    <gazebo>
        <plugin filename="gz_ros2_control-system" name="gz_ros2_control::GazeboSimROS2ControlPlugin">
        <!-- <plugin name="gazebo_ros2_control" filename="libgazebo_ros2_control.so"> -->
            <parameters>$(find dummy2-gripperv2_moveit_config)/config/ros2_controllers.yaml</parameters>
        </plugin>
    </gazebo>
    
    <gazebo reference="finger1">
        <mu1>30.0</mu1>
        <mu2>30.0</mu2>
        <kp>30000</kp>
        <kd>15</kd>
    </gazebo>
    <gazebo reference="finger2">
        <mu1>30.0</mu1>
        <mu2>30.0</mu2>
        <kp>30000</kp>
        <kd>15</kd>
    </gazebo>
    <!-- Import control_xacro -->
    <xacro:include filename="dummy2_gazebo.friction.xacro" />
    <xacro:dummy2-gripperv2_ros2_control name="GazeboSystem" initial_positions_file="$(arg initial_positions_file)"/>

</robot>

```