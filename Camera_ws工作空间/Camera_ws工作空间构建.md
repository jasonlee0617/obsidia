### 1.OAK-D-LITE功能包构建
#### 1.底层构建
```
sudo apt install ros-humble-depthai-ros
```
#### 2.源码构建
```
cd ~/Camear_ws/src
git clone -b humble https://github.com/luxonis/depthai-ros.git
```

### 2.Realsense-D435功能包构建
#### 1.安装SDK以及依赖
```
sudo apt-get install librealsense2-dkms librealsense2-utils
sudo apt-get install librealsense2-dev librealsense2-dbg
sudo apt install ros-humble-librealsense2*
```
#### 2.源码构建
```
cd ~/Camear_ws/src
git clone https://github.com/IntelRealSense/realsense-ros.git -b ros2-master
```

#### 3.Realsense-D435 相机标定
```
ros2 run camera_calibration cameracalibrator \
  --size 10x7 \
  --square 0.012 \
  --no-service-check \
  --approximate 0.1 \
  --ros-args \
  -r image:=/camera/camera/color/image_raw \
  -r camera:=/camera/camera/color
```

#### 4.Realsense-D435 相机 SDK启动
```
realsense-viewer
```
### 3.编译Camera_ws工作空间
#### 1.安装依赖
```
cd ~/Camera_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```