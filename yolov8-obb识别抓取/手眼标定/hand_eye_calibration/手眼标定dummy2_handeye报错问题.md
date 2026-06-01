### 1.编译‘==cat /etc/apt/sources.list.d/ros2.list==’出现”==cat: /etc/apt/sources.list.d/ros2.list: 没有那个文件或目录==“错误
解决办法：
🛠️ 第一步：添加 ROS 2 官方源和 GPG key
sudo apt update && sudo apt install curl gnupg lsb-release
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
🛠️ 第二步：添加 ROS 2 Humble 的源
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
🛠️ 第三步：更新系统索引
sudo apt update
