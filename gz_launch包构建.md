- [[#1.创建包|1.创建包]]
- [[#2.CMakeLists.txt文件|2.CMakeLists.txt文件]]
- [[#3.出现未找到可执行包错误|3.出现未找到可执行包错误]]
	- [[#3.出现未找到可执行包错误#3.1先检查源脚本有没有可执行权限|3.1先检查源脚本有没有可执行权限]]
	- [[#3.出现未找到可执行包错误#3.2给源脚本加执行权限|3.2给源脚本加执行权限]]

### 1.创建包
```
ros2 pkg create \
  --build-type ament_cmake \
  gz_launch
```

### 2.CMakeLists.txt文件
```
install(
  DIRECTORY
    scripts
    launch
    config
    rviz
    worlds
  DESTINATION
    share/${PROJECT_NAME}/
)

install(
  PROGRAMS
  scripts/pick_drop_ik_node.py
  scripts/pick_drop_node.py
  launch/gazebo.launch.py
  launch/gazebo_yolo.launch.py
  launch/pick_block.launch.py
  launch/yolo_pick.launch.py
  DESTINATION lib/${PROJECT_NAME}
)
ament_package()
```

### 3.出现未找到可执行包错误

#### 3.1先检查源脚本有没有可执行权限
```
ls -l ~/S622_robotarm/src/gz_launch/scripts/pick_drop_ik_node.py(可执行文件名字)
```

如果看到类似：
`-rw-r--r--`（没有 x）➡️ 就是问题根源

#### 3.2给源脚本加执行权限
```
chmod +x ~/S622_robotarm/src/gz_launch/scripts/pick_drop_ik_node.py
chmod +x ~/S622_robotarm/src/gz_launch/scripts/pick_drop_node.py
chmod +x ~/S622_robotarm/src/gz_launch/scripts/robot_control_from_UI_node.py
```

