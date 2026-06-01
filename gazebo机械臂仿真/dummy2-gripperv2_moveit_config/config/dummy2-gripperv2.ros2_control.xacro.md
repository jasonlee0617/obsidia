```
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">
    <xacro:macro name="dummy2-gripperv2_ros2_control" params="name initial_positions_file">
        <xacro:property name="initial_positions" value="${load_yaml(initial_positions_file)['initial_positions']}"/>

        <ros2_control name="${name}" type="system">
            <hardware>
                <!-- By default, set up controllers for simulation. This won't work on real hardware -->
                <plugin>mock_components/GenericSystem</plugin>
            </hardware>
            <joint name="joint1">
                <command_interface name="position"/>
                <state_interface name="position">
                  <param name="initial_value">${initial_positions['joint1']}</param>
                </state_interface>
                <state_interface name="velocity"/>
            </joint>
            <joint name="joint2">
                <command_interface name="position"/>
                <state_interface name="position">
                  <param name="initial_value">${initial_positions['joint2']}</param>
                </state_interface>
                <state_interface name="velocity"/>
            </joint>
            <joint name="joint3">
                <command_interface name="position"/>
                <state_interface name="position">
                  <param name="initial_value">${initial_positions['joint3']}</param>
                </state_interface>
                <state_interface name="velocity"/>
            </joint>
            <joint name="joint4">
                <command_interface name="position"/>
                <state_interface name="position">
                  <param name="initial_value">${initial_positions['joint4']}</param>
                </state_interface>
                <state_interface name="velocity"/>
            </joint>
            <joint name="joint5">
                <command_interface name="position"/>
                <state_interface name="position">
                  <param name="initial_value">${initial_positions['joint5']}</param>
                </state_interface>
                <state_interface name="velocity"/>
            </joint>
            <joint name="joint6">
                <command_interface name="position"/>
                <state_interface name="position">
                  <param name="initial_value">${initial_positions['joint6']}</param>
                </state_interface>
                <state_interface name="velocity"/>
            </joint>
            <joint name="figer1">
                <command_interface name="position"/>
                <state_interface name="position">
                  <param name="initial_value">${initial_positions['figer1']}</param>
                </state_interface>
                <state_interface name="velocity"/>
            </joint>
            <joint name="figer2">
                <command_interface name="position"/>
                <state_interface name="position">
                  <param name="initial_value">${initial_positions['figer2']}</param>
                </state_interface>
                <state_interface name="velocity"/>
            </joint>
        </ros2_control>
    </xacro:macro>
</robot>
```

### 关节接口配置（机械臂部分）
```
<joint name="joint1">
    <command_interface name="position"/>
    <state_interface name="position">
        <param name="initial_value">${initial_positions['joint1']}</param>
    </state_interface>
    <state_interface name="velocity"/>
</joint>

- **关节名称**：`joint1`
- **命令接口**：
    - `position`：位置控制模式
    - **作用**：接收控制器发送的位置指令
- **状态接口**：
    1. `position`：关节位置反馈
        - `initial_value`：从 YAML 加载初始值
    2. `velocity`：关节速度反馈
- **设计特点**：
    - 位置控制模式（适合大多数机械臂）
    - 提供位置和速
    - 初始位置可配置
```

### 关节接口配置（夹爪部分）
```
<joint name="figer1">
    <command_interface name="position"/>
    <state_interface name="position">
        <param name="initial_value">${initial_positions['figer1']}</param>
    </state_interface>
    <state_interface name="velocity"/>
</joint>

- **关节名称**：`figer1`
- **接口类型**：与机械臂关节相同
- **特殊考虑**：
    - 夹爪通常需要力控制，但此处使用位置控制
    - 实际应用可能需修改为：
    - <command_interface name="effort"/>  <!-- 力控制模式 -->
```

### 配置架构图解
![[Pasted image 20250803142530.png]]
