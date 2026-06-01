### 1.下载源代码
git clone https://gitee.com/switchpi/dummy2.git
### 2.编译dummy2_hand_eye_calibration
进入到dummy2 ros2 手眼标定路径下，使用
```bash
cat /etc/apt/sources.list.d/ros2.list
```
直接编辑它：
```bash
sudo nano /etc/apt/sources.list.d/ros2.list
```
然后把里面的内容改成：
```bash
deb [arch=amd64 signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu/ jammy main
```
回到dummy2_hand_eye_calibration_ws路径下，执行：
```bash
rosdep install -y -r -i --rosdistro ${ROS_DISTRO} --from-paths .
```
运行
```bash
colcon build --packages-select dummy2_interface --merge-install --symlink-install --cmake-args "-DCMAKE_BUILD_TYPE=Release"

1. **`colcon build`**：ROS2的工作空间构建命令
2.  **`--packages-select dummy2_interface`**：
    - 只构建指定的包（dummy2_interface）
    - 跳过工作空间中的其他包
3. **`--merge-install`**：
    - 将所有包的安装文件合并到单个目录（`install/`）
    - 默认行为是每个包有自己的安装子目录（`install/<package_name>`）
    - 优点：简化环境设置，减少文件重复
4. **`--symlink-install`**：
    - 使用符号链接而非复制文件到安装目录
    - 构建目录中的文件变化会实时反映到安装目录
5. **`--cmake-args "-DCMAKE_BUILD_TYPE=Release"`**：
    - 传递给CMake的额外参数
    - `-DCMAKE_BUILD_TYPE=Release`：设置构建类型为发布模式（优化执行速度）
```
接着运行
```bash
colcon build --merge-install --symlink-install --cmake-args "-DCMAKE_BUILD_TYPE=Release"

- **构建工作空间中的所有包**
- 其他参数（merge/symlink/Release）含义相同
```
#### `--merge-install`
- **工作原理**：
    # 默认模式
    install/
      ├── dummy2_interface/
      │   ├── lib/
      │   ├── share/
      ├── dummy2_hw/
      │   ├── lib/
      │   ├── share/
    # --merge-install 模式
    install/
      ├── lib/
      ├── share/
      ├── bin/
### 3.修改相应文件：
修改/dummy2/ros2/dummy2_hand_eye_calibration_ws/src/dummy2-gripperv3_description/urdf中的“dummy2-gripperv3.xacro”：
```
<joint name="world_joint" type="fixed">
  <parent link="world" />
  <child link = "base_link" />
  <origin xyz="0.0 0.0 0.0" rpy="0.0 0.0 1.575" />
  <origin xyz="0.0 0.0 0.0" rpy="0.0 0.0 0" />
</joint>
```
修改为：
```
<joint name="world_joint" type="fixed">
  <parent link="world" />
  <child link = "base_link" />
  <origin xyz="0.0 0.0 0.0" rpy="0.0 0.0 0.0" />
  <origin xyz="0.0 0.0 0.0" rpy="0.0 0.0 0" />
</joint>
```

修改dummy2/ros2/dummy2_hand_eye_calibration_ws/src/rboot_libs/rboot_libs中“rbootlibs.py”：
修改"M1-M6"减速比“'reduction': 50,”

修改“calibrate.launch.py”与“validate.launch.py”文件中：
```
    static_tf_publisher = Node(
        package="tf2_ros",
        executable="static_transform_publisher",
        arguments=["0", "0", "0", "0", "0", "0", "world", "camera_link"],
        output="screen",
    )
```
修改为：
```
    static_tf_publisher = Node(
        package="tf2_ros",
        executable="static_transform_publisher",
        arguments=["0", "0", "0", "1.5708", "0", "0", "world", "camera_link"],
        output="screen",
    )
```

如果是OAK-D-LITE相机则修改“calibrate.launch.py”与“validate.launch.py”文件中
```
    ar_moveit = IncludeLaunchDescription(
        ar_moveit_launch, launch_arguments=ar_moveit_args
    )
```
修改为：
```
    ar_moveit = IncludeLaunchDescription(
        ar_moveit_launch, 
        launch_arguments={
      "use_rviz": "true",             # 保证会 include MoveIt 的 rviz launch
      "rviz_config": rviz_config_file, # 传给 moveit_rviz.launch.py 的参数名
       }.items(),
    )
```

