此文件完整定义了机器人的物理结构和运动学特性，是仿真和控制的基础。详细的惯性参数和网格引用确保了仿真的准确性，合理的关节配置实现了6自由度机械臂+夹爪的功能。
```
<?xml version="1.0" ?>
<robot name="dummy2-gripperv2" xmlns:xacro="http://www.ros.org/wiki/xacro">

<xacro:include filename="$(find dummy2-gripperv2_description)/urdf/materials.xacro" />
<!-- <xacro:include filename="$(find dummy2-gripperv2_description)/urdf/dummy2-gripperv2.trans" /> -->
<xacro:include filename="$(find dummy2-gripperv2_description)/urdf/dummy2-gripperv2.gazebo" />
<link name="base_link">
  <inertial>
    <origin xyz="2.242591646188575e-07 0.00022711838502637176 0.0543574942352709" rpy="0 0 0"/>
    <mass value="1.2152141810431654"/>
    <inertia ixx="0.002105" iyy="0.002245" izz="0.002436" ixy="-0.0" iyz="-1.1e-05" ixz="0.0"/>
  </inertial>
  <visual>
    <origin xyz="0 0 0" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/base_link.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="0 0 0" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/base_link.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>

<link name="link1_1">
  <inertial>
    <origin xyz="-0.0056980705903348725 0.00648616835675352 0.012857190819102027" rpy="0 0 0"/>
    <mass value="0.1444690972471256"/>
    <inertia ixx="6.5e-05" iyy="5.4e-05" izz="9e-05" ixy="2.1e-05" iyz="-1.5e-05" ixz="9e-06"/>
  </inertial>
  <visual>
    <origin xyz="0.0 0.0 -0.096" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link1_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="0.0 0.0 -0.096" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link1_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>

<link name="link2_1">
  <inertial>
    <origin xyz="0.0188447417509659 0.0019829653147296275 0.08367138108787778" rpy="0 0 0"/>
    <mass value="0.6521077215074461"/>
    <inertia ixx="0.002179" iyy="0.002318" izz="0.000411" ixy="-1e-06" iyz="2.2e-05" ixz="0.000265"/>
  </inertial>
  <visual>
    <origin xyz="0.011639 -0.034477 -0.1245" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link2_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="0.011639 -0.034477 -0.1245" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link2_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>

<link name="link3_1">
  <inertial>
    <origin xyz="-0.01071065579982854 -0.027860026044143744 0.04896369644132431" rpy="0 0 0"/>
    <mass value="0.9005346433244714"/>
    <inertia ixx="0.000823" iyy="0.000697" izz="0.000535" ixy="-4e-05" iyz="0.000126" ixz="8.7e-05"/>
  </inertial>
  <visual>
    <origin xyz="-0.024511 -0.034477 -0.2925" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link3_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="-0.024511 -0.034477 -0.2925" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link3_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>

<link name="link4_1">
  <inertial>
    <origin xyz="-0.0052376144149527196 0.0600375883820652 0.0005874969147224296" rpy="0 0 0"/>
    <mass value="0.3580312221089852"/>
    <inertia ixx="0.000624" iyy="0.000202" izz="0.000688" ixy="4e-05" iyz="-5e-06" ixz="3e-06"/>
  </inertial>
  <visual>
    <origin xyz="-0.011349 -0.038577 -0.354967" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link4_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="-0.011349 -0.038577 -0.354967" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link4_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>

<link name="link5_1">
  <inertial>
    <origin xyz="-0.014439500295566867 0.07274346111500346 -0.0007483346369849819" rpy="0 0 0"/>
    <mass value="0.7858738652766984"/>
    <inertia ixx="0.00089" iyy="0.000343" izz="0.000973" ixy="0.000144" iyz="6e-06" ixz="-4e-06"/>
  </inertial>
  <visual>
    <origin xyz="-0.032649 -0.148577 -0.354967" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link5_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="-0.032649 -0.148577 -0.354967" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link5_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>

<link name="link6_1">
  <inertial>
    <origin xyz="5.087622746753444e-05 0.04245066924408897 6.811081131086194e-05" rpy="0 0 0"/>
    <mass value="0.604072405055537"/>
    <inertia ixx="0.000334" iyy="0.000382" izz="0.000521" ixy="-2e-06" iyz="2e-06" ixz="-1e-06"/>
  </inertial>
  <visual>
    <origin xyz="-0.012827 -0.268077 -0.353741" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link6_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="-0.012827 -0.268077 -0.353741" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/link6_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>

<link name="figer1_1">
  <inertial>
    <origin xyz="-0.001013816286429263 0.017871453686335137 -9.360034337241308e-05" rpy="0 0 0"/>
    <mass value="0.0422978780886348"/>
    <inertia ixx="8e-06" iyy="2e-06" izz="8e-06" ixy="1e-06" iyz="0.0" ixz="0.0"/>
  </inertial>
  <visual>
    <origin xyz="-0.025651 -0.342539 -0.353641" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/figer1_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="-0.025651 -0.342539 -0.353641" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/figer1_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>

<link name="figer2_1">
  <inertial>
    <origin xyz="0.001014115315322053 0.01787145295991821 9.418612516776115e-05" rpy="0 0 0"/>
    <mass value="0.042297878145578977"/>
    <inertia ixx="8e-06" iyy="2e-06" izz="8e-06" ixy="-1e-06" iyz="-0.0" ixz="0.0"/>
  </inertial>
  <visual>
    <origin xyz="-0.000651 -0.342539 -0.353741" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/figer2_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="-0.000651 -0.342539 -0.353741" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/figer2_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link>

<!-- <link name="camera_1">
  <inertial>
    <origin xyz="-9.097251971671924e-05 0.00021876227592271258 -0.00557732444371406" rpy="0 0 0"/>
    <mass value="0.14371269577068077"/>
    <inertia ixx="1.5e-05" iyy="0.000107" izz="0.000113" ixy="-0.0" iyz="0.0" ixz="0.0"/>
  </inertial>
  <visual>
    <origin xyz="-0.012827 -0.301289 -0.330241" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/camera_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
    <material name="silver"/>
  </visual>
  <collision>
    <origin xyz="-0.012827 -0.301289 -0.330241" rpy="0 0 0"/>
    <geometry>
      <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/camera_1.stl" scale="0.001 0.001 0.001"/>
    </geometry>
  </collision>
</link> -->

<joint name="joint1" type="revolute">
  <origin xyz="0.0 0.0 0.096" rpy="0 0 0"/>
  <parent link="base_link"/>
  <child link="link1_1"/>
  <axis xyz="0.0 -0.0 -1.0"/>
  <limit upper="3.001966" lower="-3.001966" effort="100" velocity="100"/>
</joint>

<joint name="joint2" type="revolute">
  <origin xyz="-0.011639 0.034477 0.0285" rpy="0 0 0"/>
  <parent link="link1_1"/>
  <child link="link2_1"/>
  <axis xyz="1.0 0.0 -0.0"/>
  <limit upper="1.308997" lower="-1.570796" effort="100" velocity="100"/>
</joint>

<joint name="joint3" type="revolute">
  <origin xyz="0.03615 0.0 0.168" rpy="0 0 0"/>
  <parent link="link2_1"/>
  <child link="link3_1"/>
  <axis xyz="-1.0 -0.0 -0.0"/>
  <limit upper="1.570796" lower="-1.500983" effort="100" velocity="100"/>
</joint>

<joint name="joint4" type="revolute">
  <origin xyz="-0.013162 0.0041 0.062467" rpy="0 0 0"/>
  <parent link="link3_1"/>
  <child link="link4_1"/>
  <axis xyz="0.0 -1.0 0.0"/>
  <limit upper="3.141593" lower="-3.141593" effort="100" velocity="100"/>
</joint>

<joint name="joint5" type="revolute">
  <origin xyz="0.0213 0.11 0.0" rpy="0 0 0"/>
  <parent link="link4_1"/>
  <child link="link5_1"/>
  <axis xyz="-1.0 -0.0 -0.0"/>
  <limit upper="2.094395" lower="-1.919862" effort="100" velocity="100"/>
</joint>

<joint name="joint6" type="revolute">
  <origin xyz="-0.019822 0.1195 -0.001226" rpy="0 0 0"/>
  <parent link="link5_1"/>
  <child link="link6_1"/>
  <axis xyz="0.0 -1.0 -0.0"/>
  <limit upper="3.141593" lower="-3.141593" effort="100" velocity="100"/>
</joint>

<joint name="figer1" type="prismatic">
  <origin xyz="0.012824 0.074462 -0.0001" rpy="0 0 0"/>
  <parent link="link6_1"/>
  <child link="figer1_1"/>
  <axis xyz="1.0 -0.0 0.0"/>
  <limit upper="0.028" lower="0.0" effort="100" velocity="100"/>
</joint>

<joint name="figer2" type="prismatic">
  <origin xyz="-0.012176 0.074462 0.0" rpy="0 0 0"/>
  <parent link="link6_1"/>
  <child link="figer2_1"/>
  <axis xyz="1.0 -0.0 0.0"/>
  <limit upper="0.0" lower="-0.028" effort="100" velocity="100"/>
  <!-- <mimic joint="figer1" multiplier="-1" offset="0"/> -->
</joint>

<!-- <link name="grasp_frame">
  <visual>
    <geometry><sphere radius="0.01"/></geometry>
    <material name="green"><color rgba="0 1 0 1"/></material>
  </visual>
  <collision>
    <geometry><sphere radius="0.005"/></geometry>
  </collision>
</link>

<joint name="grasp_frame_joint" type="fixed">
  <origin xyz="0 -0.06 0" rpy="0 0 0"/>
  <parent link="link6_1"/>
  <child link="grasp_frame"/>
</joint> -->
<link name="grasp_frame">
  <!-- 添加质量属性以保持一致性 -->
  <!-- <inertial>
    <mass value="0.008"/>
    <inertia ixx="1e-9" iyy="1e-9" izz="1e-9" ixy="0" iyz="0" ixz="0"/>
  </inertial> -->
  <!-- 可视化标记：绿色小球便于在RViz中识别（gazebo仿真中为避免碰撞需注释掉） -->
  <!-- <visual>
    <geometry>
      <sphere radius="0.008"/>
    </geometry>
    <material name="grasp_frame_green">
      <color rgba="0 1 0 0.8"/>
    </material>
  </visual> -->
</link>

<joint name="grasp_frame_joint" type="fixed">
  <origin xyz="0.0 0.095 0.0" rpy="0 0 0"/>
  <parent link="link6_1"/>
  <child link="grasp_frame"/>
</joint>


</robot>

```

### 1. XML 声明和根元素
```
<?xml version="1.0" ?>
<robot name="dummy2-gripperv2" xmlns:xacro="http://www.ros.org/wiki/xacro">

- **XML 声明**：标准 XML 文件头
- **根元素**：`<robot>` 定义机器人模型
- **机器人名称**：`name="dummy2-gripperv2"` 与顶层文件一致
- **命名空间**：`xmlns:xacro` 启用 Xacro 宏功能
```

### 2. 包含外部文件
```
<xacro:include filename="$(find dummy2-gripperv2_description)/urdf/materials.xacro" />
- **材料定义**：包含材料属性文件
- **内容**：应定义颜色材质（如 `silver`）
```

```
<xacro:include filename="$(find dummy2-gripperv2_description)/urdf/dummy2-gripperv2.trans" />
- **Gazebo 扩展**：包含 Gazebo 仿真专用配置
- **内容**：物理引擎参数、传感器插件等
```

### 3. 基座连杆 (base_link)
```
<link name="base_link">
- **基础连杆**：机器人固定基座
```

```
<inertial>
  <origin xyz="2.242591646188575e-07 0.00022711838502637176 0.0543574942352709" rpy="0 0 0"/>
  <mass value="1.2152141810431654"/>
  <inertia ixx="0.002105" iyy="0.002245" izz="0.002436" ixy="-0.0" iyz="-1.1e-05" ixz="0.0"/>
</inertial>

- **惯性参数**：
    - `origin`：质心位置 (x,y,z) 和姿态 (roll,pitch,yaw)
    - `mass`：质量 1.215 kg
    - `inertia`：惯性张量矩阵
```

```
<visual>
  <origin xyz="0 0 0" rpy="0 0 0"/>
  <geometry>
    <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/base_link.stl" scale="0.001 0.001 0.001"/>
  </geometry>
  <material name="silver"/>
</visual>

- **视觉属性**：
    - `mesh`：STL 网格文件路径
    - `scale`：毫米转米 (0.001)
    - `material`：引用 silver 材质
```

```
<collision>
  <origin xyz="0 0 0" rpy="0 0 0"/>
  <geometry>
    <mesh filename="file://$(find dummy2-gripperv2_description)/meshes/base_link.stl" scale="0.001 0.001 0.001"/>
  </geometry>
</collision>
</link>
- **碰撞属性**：与视觉模型相同（可简化以提高性能）
```

### 4. 机械臂连杆 (link1_1 到 link6_1)
```
<link name="link1_1">...</link>
<link name="link2_1">...</link>
<link name="link3_1">...</link>
<link name="link4_1">...</link>
<link name="link5_1">...</link>
<link name="link6_1">...</link>

- **结构相同**：每个连杆包含：
    1. 惯性参数（质心、质量、惯性张量）
    2. 视觉属性（网格、材质）
    3. 碰撞属性（与视觉相同）
- **参数特点**：
    - 质量递减：从基座到末端
    - 质心位置反映几何形状
    - 惯性张量基于几何计算
```

### 5. 夹爪连杆 (figer1_1, figer2_1)
```
<link name="figer1_1">...</link>
<link name="figer2_1">...</link>
- **对称设计**：
    - 质量相同：0.042 kg
    - 惯性张量相似
    - 位置镜像对称
- **末端执行器**：抓取物体的关键部分
```

### 6. 相机连杆 (camera_1)
```
<link name="camera_1">...</link>

- **视觉传感器**：位于末端
- **质量**：0.143 kg
- **位置**：连接在 link6_1 上
```

### 7. 旋转关节 (Revolute Joints)
```
<joint name="joint1" type="revolute">
  <origin xyz="0.0 0.0 0.096" rpy="0 0 0"/>
  <parent link="base_link"/>
  <child link="link1_1"/>
  <axis xyz="0.0 -0.0 -1.0"/>
  <limit upper="3.001966" lower="-3.001966" effort="100" velocity="100"/>
</joint>

- **类型**：`revolute`（旋转关节）
- **运动链**：
    - `parent`：父连杆
    - `child`：子连杆
        
- **几何关系**：
    - `origin`：相对于父连杆的位置和姿态
    - `axis`：旋转轴方向
- **运动限制**：
    - `upper/lower`：角度限制（弧度）
    - `effort`：最大扭矩（Nm）
    - `velocity`：最大速度（rad/s）
```

### 8. 平移关节 (Prismatic Joints)
```
<joint name="figer1" type="prismatic">
  <origin xyz="0.012824 0.074462 -0.0001" rpy="0 0 0"/>
  <parent link="link6_1"/>
  <child link="figer1_1"/>
  <axis xyz="1.0 -0.0 0.0"/>
  <limit upper="0.028" lower="0.0" effort="100" velocity="100"/>
</joint>

- **类型**：`prismatic`（平移关节）
- **夹爪设计**：
    - `upper="0.028"`：最大张开距离 28mm
    - `lower="0.0"`：完全闭合位置
```

```
<joint name="figer2" type="prismatic">
  <limit upper="0.0" lower="-0.028"/>
</joint>
- 负向运动实现对称开合
```

### 9. 相机固定关节
```
<joint name="slider_camera" type="prismatic">
  <limit upper="0.0" lower="0.0" effort="100" velocity="100"/>
</joint>
- **实际固定**：上下限均为0，表示不可移动
- **类型选择**：虽声明为平移关节，但实际是固定连接
```

### 10. 文件结尾
```
</robot>
- 关闭根元素
```

### 机器人结构总结
base_link → joint1 → link1_1 → joint2 → link2_1 → ... → link6_1
               ├─ figer1 → figer1_1 (夹爪指1)
               ├─ figer2 → figer2_1 (夹爪指2)
               └─ slider_camera → camera_1 (摄像头)