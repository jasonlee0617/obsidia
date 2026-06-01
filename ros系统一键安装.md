### 1.安装共享粘贴
#### 安装 `open-vm-tools`
##### **卸载旧版本工具**
sudo apt autoremove open-vm-tools
##### **安装基础工具及桌面组件**：
sudo apt update
sudo apt install open-vm-tools
sudo apt install open-vm-tools-desktop
##### **重启虚拟机**：
sudo reboot
#### 2. **启用共享剪贴板设置**
- 关闭Ubuntu虚拟机。
- 在VMware中：  
    ==`虚拟机设置 → 选项 → 客户机隔离`== → 勾选 启用复制粘贴。

### 2.ros一键安装
#### 一键配置系统源
  打开ubuntu终端，输入：
  wget http://fishros.com/install -O fishros && . fishros
  然后会出现以下界面：
  ![[Pasted image 20250728204456.png]]
  接着输入数字5，回车，一键配置系统源
  ![[Pasted image 20250728204637.png]]
  输入数字2，回车，更换系统源并清理第三方源
  ![[Pasted image 20250728204728.png]]
  输入数字1，回车，添加ros/ros2源

### 配置ubuntu相应版本
打开ubuntu终端，输入：
wget http://fishros.com/install -O fishros && . fishros

然后我们输入 **1** 一键安装
![[Pasted image 20250728205005.png]]
不更换源安装
![[Pasted image 20250728205041.png]]
选择相应的ubuntu版本对应的ros版本
![[Pasted image 20250728205128.png]]

### 配置rosdep
打开ubuntu终端，输入：
wget http://fishros.com/install -O fishros && . fishros
再输入 **3** 就一键配置

### 更新系统环境
打开ubuntu终端，输入：
wget http://fishros.com/install -O fishros && . fishros
再输入 **4** 就一键配置

### ros2-jazzy的一键安装
#### 启用Ubuntu Universe 仓库，两条命令如下：
sudo apt install software-properties-common -y 
sudo add-apt-repository universe
![[Pasted image 20250729152455.png]]
#### 下载ROS 2 Jazzy安装工具，命令如下：
wget https://github.com/ros-infrastructure/ros-apt-source/releases/download/1.1.0 /ros2-apt-source_1.1.0.noble_all.deb -nv
#### 安装ROS 2 Jazzy安装工具，命令如下：
sudo apt install ./ros2-apt-source_1.1.0.noble_all.deb
#### 更新软件列表和升级应用，两条命令如下：
sudo apt update 
sudo apt upgrade -y
![[Pasted image 20250729152634.png]]
#### 安装ROS 2 Jazzy的开发工具，命令如下：
sudo apt install ros-dev-tools -y
![[Pasted image 20250729152732.png]]

#### 为了方便后续在打开终端后直接使用ROS 2的 相关命令，需要将ROS 2的配置文件添加到.bashrc文件中，并且在当前的终端重新加载配 置，以启用ROS 2的命令。在终端里执行以下两条命令：
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc source ~/.bashrc
#### 验证ROS 2 Jazzy，命令如下：
ros2 run turtlesim turtlesim_node
ros2 run turtlesim turtle_teleop_key
![[Pasted image 20250729152840.png]]
