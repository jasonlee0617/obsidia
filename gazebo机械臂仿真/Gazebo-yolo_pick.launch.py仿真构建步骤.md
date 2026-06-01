### 1.克隆项目
```
git clone --recurse-submodules https://github.com/laoxue888/Moveit2YoloObb.git
```
### 2.配置开发环境
#### 2.1.为graph_executer配置虚拟环境
##### 2.1.1创建虚拟环境

```
##创建虚拟环境
python3 -m venv ~/Moveit2YoloObb/GraphExecuter/graph_executer
##激活虚拟环境
source ~/Moveit2YoloObb/GraphExecuter/graph_executer/bin/activate
##创建虚拟环境
python3 -m venv ~/S622_robotarm/src/GraphExecuter/graph_executer
##激活虚拟环境
source ~/S622_robotarm/src/GraphExecuter/graph_executer/bin/activate
##退出虚拟环境
deactivate

##删除虚拟环境
rm -rf bin lib lib64 include pyvenv.cfg

```

创建一个 constraints：
```
cd graph_executer
cat > constraints.txt <<'EOF'
numpy==1.23.0
opencv-python==4.6.0.66
opencv-python-headless==4.6.0.66
transforms3d==0.4.2
EOF
```

##### 2.1.2下载相关依赖

```
cd graph_executer
pip install -r requirements_linux.txt -c constraints.txt
git clone https://github.com/laoxue888/NodeGraphQt.git
cd NodeGraphQt
pip install -e . -c constraints.txt
cd graph_executer
pip install torch==2.4.1 torchvision==0.19.1 torchaudio==2.4.1 --index-url https://download.pytorch.org/whl/cu118 -c constraints.txt
sudo apt update
sudo apt install -y portaudio19-dev x11-xserver-utils
sudo apt install -y libturbojpeg
sudo apt install -y "libxcb*"


pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple -c constraints.txt 

pip install jpeg4py -i https://pypi.tuna.tsinghua.edu.cn/simple -c constraints.txt 
sudo apt-get install -y libturbojpeg
```

##### 2.1.3错误修复

[21:32:44]: Error importing nodes.panda_arm.arm_control: No module named 'panda_arm_msg'
[21:32:44]: Error importing nodes.panda_arm.unity_arm: No module named 'panda_arm_msg'
[21:32:44]: Error importing nodes.speach.speech: No module named 'pyaudio'

先装系统依赖：
```
sudo apt update
sudo apt install -y portaudio19-dev python3-dev build-essential
```

再在 **venv** 里装：
```
python -m pip install -U pip setuptools wheel
python -m pip install pyaudio -c constraints.txt
python -m pip install ultralytics==8.3.217 -c constraints.txt
```
#### 2.2 ros_project工作空间构建.
##### 2.2.1 下载yolo依赖
```
pip install ultralytics==8.3.217
pip show ultralytics
```
##### 2.2.2 dummy2_gazebo工作空间构建
###### 2.2.2.1
```
ros2 pkg create --build-type ament_python dummy2_gazebo --dependencies rclpy 
```
###### 2.2.2.2 setup.py内容修改

1.dummy2_gazebo包下创建config、launch、rviz、worlds目录
其中worlds文件直接复制panda_moveit_config/worlds

2.修改setup.py中'data_files'内容：
```
 ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
        (os.path.join('share', package_name, 'config'), glob('config/*.yaml')),
        (os.path.join('share', package_name, 'rviz'), glob('rviz/*.rviz')), 
        (os.path.join('share', package_name, 'config'), glob('config/*.sdf')),
        (os.path.join('share', package_name, 'config'), glob('config/*.urdf')),
        (os.path.join('share', package_name, 'config'), glob('config/*.xacro')),
        (os.path.join('share', package_name, 'config'), glob('config/*.srdf')),
        (os.path.join('share', package_name, 'config'), glob('config/*.urdf.xacro')),
        (os.path.join('share', package_name, 'config'), glob('config/*.xacro')),
        # (os.path.join('share', package_name, 'scripts'), glob('scripts/*.py')),
        (os.path.join('share', package_name, 'worlds', 'bolt', 'meshes'), glob('worlds/bolt/meshes/*')),
        # (os.path.join('share', package_name, 'worlds', 'bolt'), glob('worlds/bolt/*')),
        (os.path.join('share', package_name, 'worlds', 'bolt'), [f for f in glob('worlds/bolt/*') if os.path.isfile(f)]),
        (os.path.join('share', package_name, 'worlds', 'ground_plane'), glob('worlds/ground_plane/*')),
        (os.path.join('share', package_name, 'worlds', 'sun'), glob('worlds/sun/*')),
        (os.path.join('share', package_name, 'worlds', 'table'), glob('worlds/table/*')),
        # 包含 arm_on_the_table.sdf 文件
        (os.path.join('share', package_name, 'worlds'), ['worlds/arm_on_the_table.sdf']),
```

3.添加控制控制节点，修改'console_scripts': 
```
            'pick_drop = dummy2_gazebo.pick_drop_node:main',
            'pick_drop_ik = dummy2_gazebo.pick_drop_ik_node:main',
            'dummy2_control_from_UI = dummy2_gazebo.dummy2_control_from_UI:main',
```

###### 2.2.2.3 gazebo_yolo.launch.py的修改

1.复制panda_moveit_config/launch/gazebo.launch.py为gazebo_yolo.launch.py

2.修改部分代码

```
gazebo_node = IncludeLaunchDescription(
                PythonLaunchDescriptionSource([os.path.join(get_package_share_directory('ros_gz_sim'), 'launch'), '/gz_sim.launch.py']),
                launch_arguments=[
                    ('gz_args', [LaunchConfiguration('world'),
                                 '.sdf',
                                 ' -v 4',
                                 ' -r',
                                 ' --physics-engine gz-physics-bullet-featherstone-plugin']
                    )
                ]
             )
```
==替换为==
```
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
```

```
    moveit_config =(MoveItConfigsBuilder("panda_arm", package_name="panda_moveit_config")
                .robot_description('config/panda.urdf.xacro')
                .moveit_cpp(arm_robot_sim_path + "/config/controller_setting.yaml") # moveit settings
                .robot_description_semantic('config/panda.srdf').to_moveit_configs()
    )
```
==替换为==
```
    moveit_config =(MoveItConfigsBuilder("dummy2-gripperv2", package_name="dummy2-gripperv2_moveit_config")
                .robot_description(packagepath + '/config/dummy2_gazebo.urdf.xacro')
                .moveit_cpp(arm_robot_sim_path + "/config/controller_setting.yaml") # moveit settings
                .robot_description_semantic('config/dummy2-gripperv2.srdf').to_moveit_configs()
    )
```

3.moveit_config、description文件==路径、名称==要相应替换

###### 2.2.2.4 dummy2_gazebo.urdf.xacro文件的创建

1.复制panda_moveit_config/config/panda.urdf.xacro文件，新建dummy2_gazebo.urdf.xacro文件

2.修改部分内容
```
 <xacro:arg name="initial_positions_file" default="$(find panda_moveit_config)/config/initial_positions.yaml"/>
```
替换为
```
<xacro:arg name="initial_positions_file" default="/home/jasonlee/Moveit2YoloObb/ros2_project/src/dummy2-gripperv2_moveit_config/config/initial_positions.yaml"/>
```

```
<xacro:include filename="$(find robot_description)/urdf/panda.urdf"/>
```
替换为
```
 <xacro:include filename="$(find dummy2-gripperv2_description)/urdf/camera/camera.xacro"/>
```

```
 <parameters>$(find panda_moveit_config)/config/ros2_controllers.yaml</parameters>
```
替换为
```
   <parameters>$(find dummy2-gripperv2_moveit_config)/config/ros2_controllers.yaml</parameters>
```

```
 <xacro:include filename="panda.gazebo.ros2_control.xacro" />
    <xacro:panda_gazebo_ros2_control name="GazeboSimSystem"
        initial_positions_file="$(arg initial_positions_file)"/>
```
替换为
```
   <xacro:include filename="dummy2_gazebo.friction.xacro" />
    <xacro:dummy2-gripperv2_ros2_control name="GazeboSimSystem" initial_positions_file="$(arg initial_positions_file)"/>
```


##### 2.2.3 dummy2-gripperv2_description文件夹修改

1.复制robot_description路径下==worlds==文件夹到dummy2-gripperv2_description/urdf
创建文件夹dummy2-gripperv2_description/urdf/worlds

2.复制robot_description/meshes路径下的==visual文件夹==中的==camera.dae==文件，创建文件夹dummy2-gripperv2_description/visual/camera.dae

3.复制robot_description/meshes路径下的==collision文件夹==中的==camera.stl==文件到dummy2-gripperv2_description/meshse

3.修改set.up文件中的‘data_files’
```
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
        # (os.path.join('share', package_name, 'urdf'), glob('urdf/*')),
        (os.path.join('share', package_name, 'urdf', 'camera'), glob('urdf/camera/*')),
        (os.path.join('share', package_name, 'urdf'), [f for f in glob('urdf/*') if os.path.isfile(f)]),
        (os.path.join('share', package_name, 'meshes'), glob('meshes/*')),
        (os.path.join('share', package_name, 'config'), glob('config/*')),
        (os.path.join('share', package_name, 'visual'), glob('visual/*')),
```

### 3.运行项目

#### 3.1构建工作空间~/Moveit2YoloObb/ros2_project

```
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
```
#### 3.2创建控制节点dummy2_control_from_UI.py文件
[[dummy2_control_from_UI.py]]

#### 3.3 创建运行脚本
[[gazebo_yolo.launch.py]]

#### 3.4运行yolo_pick.launch.py
```
source install/setup.bash
ros2 launch dummy2_gazebo yolo_pick.launch.py 
```

若显示==yolov8_obb_publisher.py==系统中找不到
```
chmod +x ~/Moveit2YoloObb/ros2_project/install//Moveit2YoloObb/ros2_project/install/yolov8_obb/lib/yolov8_obb/yolov8_obb_publisher.py
```

#### 3.5运行python3 main.py
```
cd ~/Moveit2YoloObb/GraphExecuter/graph_executer
source ~/Moveit2YoloObb/ros2_project/install/setup.bash
python3 main.py
```

点击"Tab"键添加节点
![[Pasted image 20260112223920.png]]

持久化集成终端：
```
nano ~/.bashrc
source /opt/ros/humble/setup.bash
source ~/Moveit2YoloObb/ros2_project/install/setup.bash
source ~/.bashrc
```

### 4.gazebo环境解决
![[Pasted image 20251104165118.png]]
#### 4.1修改worlds/table/model.sdf
路径：/Moveit2YoloObb/ros2_project/src/dummy2_gazebo/worlds/table/model.sdf

```
    <!-- <visual name="front_left_leg">
        <pose>0.68 0.38 0.5 0 0 0</pose>
        <geometry>
          <cylinder>
            <radius>0.02</radius>
            <length>1.0</length>
          </cylinder>
        </geometry>
        <material>
          <script>
            <uri>file://media/materials/scripts/gazebo.material</uri>
            <name>Gazebo/Grey</name>
          </script>
        </material>
      </visual>
```

替换为手写代码部分

```
<visual name="front_left_leg">  
        <pose>0.68 0.38 0.5 0 0 0</pose>  
        <geometry>  
          <cylinder>  
            <radius>0.02</radius>  
            <length>1.0</length>  
          </cylinder>  
        </geometry>  
        <material>  
          <!-- 手写 Gazebo/Grey：精确匹配 -->  
          <ambient>0.75 0.75 0.75 1</ambient>  
          <diffuse>0.75 0.75 0.75 1</diffuse>  
          <specular>0.18 0.18 0.18 1</specular>  
          <shininess>24</shininess>  
          <emissive>0 0 0 1</emissive>  
        </material>  
      </visual>  
```

#### 4.2修改worlds/ground_plane/model.sdf

```
  <visual name="visual">
        <cast_shadows>false</cast_shadows>
        <geometry>
          <plane>
            <normal>0 0 1</normal>
            <size>100 100</size>
          </plane>
        </geometry>
        <material>
          <script>
             <uri>file://media/materials/scripts/gazebo.material</uri>
            <name>Gazebo/Grey</name>
          </script>
        </material>
      </visual>
```

替换为手写代码部分：

```
<visual name="visual">  
        <cast_shadows>false</cast_shadows>  
        <geometry>  
          <plane>  
            <normal>0 0 1</normal>  
            <size>100 100</size>  
          </plane>  
        </geometry>  
        <material>  
          <!-- 传统 OGRE 风格：精确匹配 Gazebo/Grey -->  
          <ambient>0.8 0.8 0.8 1</ambient>  
          <diffuse>0.8 0.8 0.8 1</diffuse>  
          <specular>0.1 0.1 0.1 1</specular>  
          <emissive>0 0 0 0</emissive>  
          <!-- 双面渲染，确保平面无黑边 -->  
          <double_sided>true</double_sided>  
        </material>  
      </visual>  
```

#### 4.3替换效果
![[Pasted image 20251104165708.png]]
