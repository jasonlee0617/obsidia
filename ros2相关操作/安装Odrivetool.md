### 1.**确认 ODrive 固件版本**
sudo pip3 install odrive0.5.3.post0  # 需 root 权限以自动配置 udev 规则:cite[1]
### 2.**解决 USB 权限问题**
sudo cp /usr/lib/udev/rules.d/60-odrive.rules /etc/udev/rules.d/  # 复制规则文件
sudo udevadm control --reload-rules && sudo udevadm trigger      # 重载规则

### 3.**克隆正确的 ODrive ROS2 仓库**
git clone https://github.com/Factor-Robotics/odrive_ros2_control.git
### 4.**配置工作空间**
mkdir -p ~/odrive_ws/src
cp -r odrive_ros2_control ~/odrive_ws/src/
### 5.**安装 ROS 依赖**
cd ~/odrive_ws
rosdep install --from-paths src --ignore-src -r -y
sudo apt install ros-humble-can-msgs ros-humble-hardware-interface
### 6.**编译工作空间**
colcon build --symlink-install
source install/setup.bash
### USB 权限配置 (重要!)
# 创建 udev 规则
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="1209", ATTR{idProduct}=="0d[0-9][0-9]", MODE="0666"' | sudo tee /etc/udev/rules.d/91-odrive.rules

# 重新加载规则
sudo udevadm control --reload-rules
sudo udevadm trigger

odrivetool
>>> dev0.vbus_voltage  # 应返回电压值
>>> dev0.axis0.motor.config.current_lim ##电流限制
>>> dev0.axis0.motor.config.requested_current_range
>>> dev0.axis0.motor.config.torque_constant

设置电流限制并保存：
dev0.axis0.motor.config.current_lim = 40
dev0.axis0.motor.config.requested_current_range = 40
dev0.axis0.motor.config.torque_constant = 8.23 / 320
dev0.save_configuration()