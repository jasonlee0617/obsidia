### 1.建立标定板坐标系
#### 1.solidworks建立实际标定板坐标系
![[Pasted image 20260109215038.png]]
#### 2.实际标定板位置（大概）
要求标定板中的标定码中心位置与建立的标定板坐标系大致对得上
![[Pasted image 20260109215815.png]]
#### 3.获得标定板坐标系的实际位置
##### 3.1从solidworks的urdf导出插件获取标定板坐标系位置
![[Pasted image 20260109220309.png]]
##### 3.2补充机械臂的urdf文件
```
 <link name="calibration_marker"/>
  <joint name="wrist3_to_calibration_marker" type="fixed">
    <parent link="wrist3_link"/>
    <child link="calibration_marker"/>
    <origin xyz="4.8765E-05 -0.011601 0.1636" rpy="0 0 0"/>
  </joint>
```

### 4.标定过程
#### 4.1补充标定参数
```
calibration_args = {
        "name": "robot_calibration",
        "calibration_type": "eye_on_base",
        "robot_base_frame": "base_link",
        "robot_effector_frame": "calibration_marker",
        "tracking_base_frame": "camera_color_optical_frame",
        "tracking_marker_frame": "calibration_aruco",
    }
    
确认标定码的大小：
  aruco_params = os.path.join(
        get_package_share_directory("hand_eye_calibration"),
        "config",
        "aruco_parameters.yaml",
    )
aruco_parameters.yaml文件：
/aruco_node:
  ros__parameters:
    marker_size: 0.07 #单位cm
    aruco_dictionary_id: DICT_5X5_250
    image_topic: /camera/camera/color/image_raw
    camera_info_topic: /camera/camera/color/camera_info
```

#### 4.2标定流程
- **近距离**：标定码在画面里 **很大**
- **中距离**：标定码在画面里 **中等**
- **远距离**：标定码在画面里 **较小**

**每个距离层：8–10 张**
 位置覆盖
- 中心：2–3 张
- 四角里选 2–3 个角（每次平移大概10cm，平移过程中尽量做点旋转）
    
姿态变化
- Ry 大倾角（±20°～±40°）至少 3 张
- Rx 大倾角（±20°～±40°）至少 2 张
- Rz 旋转（30°/60°/90°）至少 1–2 张

**每次采集都要停留0.5-1S后再采集**
**若“Take Sample”按钮闪烁，则姿态要舍弃**
![[Pasted image 20260109221503.png]]
##### 4.2.1近距离层
**样本一（画面中心）**
![[Pasted image 20260109221530.png]]

**样本二：旋转rx轴（画面中心）**
![[Pasted image 20260109221711.png]]

**样本三：旋转+ry,+rz轴（画面中心）**
![[Pasted image 20260109221854.png]]

**样本四：旋转-ry,-rz轴，与样本三形成对称（画面中心）**
![[Pasted image 20260109222039.png]]
 
 **样本五：旋转-rx，+rz轴（画面中心）**
 ![[Pasted image 20260109222246.png]]

**样本六：平移x,z轴，微旋转+rx轴（画面左下角）**
![[Pasted image 20260109222445.png]]

**样本七：旋转+ry轴，+rz轴（画面左下角）**
![[Pasted image 20260109222634.png]]

**样本八：平移z，+rx,+ry轴(画面左上角)**
![[Pasted image 20260109222923.png]]
**样本九：+rx,+rz轴(画面左上角)**
![[Pasted image 20260109223023.png]]
**样本十：平移x,rx，ry轴(画面右上角)**
![[Pasted image 20260109223226.png]]

**样本十一：移动x,rx，ry，rz轴(画面右上角)**
![[Pasted image 20260109223403.png]]

**样本十二：移动rx，ry，rz轴(画面右上角)**
![[Pasted image 20260109223459.png]]

**样本十三：移动z,rx轴(画面右下角)**
![[Pasted image 20260109223605.png]]

**样本十四：移动rx，ry,rz轴(画面右下角)**
![[Pasted image 20260109223743.png]]

##### 4.2.2中距离层和远距离层（距离层的区别主要是标定板占画面的比列，距离可通过移动x,y轴实现，我这里的例子主要是通过y轴实现）

##### 4.2.3最终标定成功主要通过xyz的中的数据基本不再变化来判断

##### 4.2.4标定成功点击“save”保存标定数据

##### 4.2.5发布标定数据
通过**handeye_publisher.py**文件发布标定数据，后续视觉抓取或者视觉避障中要进行
**base_link -》camera**坐标转化