清除opotmap体素
ros2 service call /clear_octomap std_srvs/srv/Empty {}

# ServoStatus
```
# Message
int8 code # will contains integer code
string message # will contain explanatory message

# Status types (should reflect StatusCode from moveit_servo/utils/datatype.hpp)
int8 INVALID = -1
int8 NO_WARNING = 0
int8 DECELERATE_FOR_APPROACHING_SINGULARITY = 1
int8 HALT_FOR_SINGULARITY = 2
int8 DECELERATE_FOR_LEAVING_SINGULARITY = 3
int8 DECELERATE_FOR_COLLISION = 4
int8 HALT_FOR_COLLISION = 5
int8 JOINT_BOUND = 6
```

伺服相关依赖下载：
```
sudo tee /etc/apt/sources.list.d/ros2-testing.list > /dev/null <<'EOF'
deb [arch=amd64 signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2-testing/ubuntu jammy main
EOF
sudo apt update

apt-cache madison ros-humble-moveit-msgs
sudo apt install --only-upgrade \
  ros-humble-moveit-msgs \
  ros-humble-moveit-core \
  ros-humble-moveit-ros-planning \
  ros-humble-moveit-ros-move-group \
  ros-humble-moveit-servo


dpkg -L ros-humble-moveit-msgs | egrep 'ServoStatus|ServoCommandType|ChangeDriftDimensions'
ros2 interface show moveit_msgs/srv/ChangeDriftDimensions
```
