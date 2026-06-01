---

excalidraw-plugin: parsed
tags: [excalidraw]

---
==⚠  Switch to EXCALIDRAW VIEW in the MORE OPTIONS menu of this document. ⚠== You can decompress Drawing data with the command palette: 'Decompress current Excalidraw file'. For more info check in plugin settings under 'Saving'


# Excalidraw Data

## Text Elements
#!/usr/bin/env python ^9SASLYuf

# -*- coding: utf-8 -*- ^fBEo9WHB

from __future__ import division ^aQaEG40i

import rospy ^54UYLPNO

import message_filters ^88yzkLzd

import numpy as np ^dJpFUQKF

import cv2 ^Ej9tBLyd

from matplotlib import pyplot as plt ^r5j0AmUM

from cv_bridge import CvBridge ^oWPz7Ygh

from sensor_msgs.msg import Image ^GWwluISc

from simple_follower.msg import position as PositionMsg ^WbOrvf9T

from std_msgs.msg import String as StringMsg ^MMVqYQ5q

from std_msgs.msg import Int8 ^jFrnIazI

from dynamic_reconfigure.server import Server ^KhfbMxP3

from simple_follower.cfg import Params_colorConfig ^W78vk02S

np.seterr(all='raise') ^olanq3jX

displayImage = False ^74iLWlpI

plt.close('all') ^mEL3z95q

class visualTracker: ^vLMRHgp9

def __init__(self): ^Q3ZbBqSd

self.bridge = CvBridge() ^dr6LAoKr

self.i=0      #语音识别标志 ^TdJAuDC7

self.color_obj = Server(Params_colorConfig,self.colorreconfigure) ^mEYk8LDO

self.targetUpper = np.array(rospy.get_param('~targetred/upper')) ^MP7N6EIx

self.targetLower = np.array(rospy.get_param('~targetred/lower')) ^M7zysAIp

self.col_red_U =np.array(rospy.get_param('~targetred/upper'))# red ^nyNC6h7g

self.col_red_L =np.array(rospy.get_param('~targetred/lower')) ^6di6qGbA

self.col_blue_U =np.array(rospy.get_param('~targetblue/upper'))# blue ^S5l8kNBe

self.col_blue_L =np.array(rospy.get_param('~targetblue/lower')) ^7wO7dkT2

self.col_green_U =np.array(rospy.get_param('~targetgreen/upper'))# green ^ZNb6YwQy

self.col_green_L = np.array(rospy.get_param('~targetgreen/lower')) ^TmK3DhlD

self.col_yellow_U =np.array(rospy.get_param('~targetyellow/upper')) #yellow ^9PiAvfYw

self.col_yellow_L =np.array(rospy.get_param('~targetyellow/lower')) ^hGVevpz9

self.pictureHeight= rospy.get_param('~pictureDimensions/pictureHeight') ^c33IiNg1

self.pictureWidth = rospy.get_param('~pictureDimensions/pictureWidth') ^J3z6wr7M

vertAngle =rospy.get_param('~pictureDimensions/verticalAngle') ^wIMmuW0X

horizontalAngle =  rospy.get_param('~pictureDimensions/horizontalAngle') ^jvmtBqEb

# precompute tangens since thats all we need anyways: 预计算切线 ^QV7dwnht

self.tanVertical = np.tan(vertAngle) ^EoGt8hf0

self.tanHorizontal = np.tan(horizontalAngle) ^Tw0V1EYn

self.lastPoCsition =None ^Rbo73o7Z

self.targetDist = rospy.get_param('~targetDist') ^EKoh9C7z

# one callback that deals with depth and rgb at the same time
 一个同时处理深度和RGB的回调 ^78q5MSKq

im_sub = message_filters.Subscriber('/camera/rgb/image_raw', Image) ^SYn1CzoY

dep_sub = message_filters.Subscriber('/camera/depth/image_raw', Image) ^nIVO8gSC

self.timeSynchronizer = message_filters.ApproximateTimeSynchronizer([im_sub, dep_sub], 10, 0.5) ^9AcMfi2G

self.timeSynchronizer.registerCallback(self.trackObject) ^Qj5j1nDd

self.positionPublisher = rospy.Publisher('/object_tracker/current_position', PositionMsg, queue_size=3) ^qUpiTLDv

#self.infoPublisher = rospy.Publisher('/object_size=3) ^x4h4l1ju

rospy.logwarn(self.targetUpper) ^J4Ng0mgu

def publish_flag(self): ^qqkKZx7D

visual_follow_flag=Int8() ^oNDLiT6r

visual_follow_flag.data=1 ^HwMQcHqZ

rospy.sleep(1.) ^rBHeTUTn

visualfwflagPublisher.publish(visual_follow_flag) ^Y3adhmVl

rospy.loginfo('a=%d',visual_follow_flag.data) ^Di697NNu

print("1111111111111111111111111111111111111111111111111111111111111") ^sIs9aysP

def trackObject(self, image_data, depth_data): ^YYSUqOG1

if(image_data.encoding != 'rgb8'): ^g0yCoBDr

raise ValueError('image is not rgb8 as expected') ^a8ka2PrO

#convert both images to numpy arrays # 将两张图片都转换为Numpy数组 ^ZcmuBX9q

frame = self.bridge.imgmsg_to_cv2(image_data, desired_encoding='rgb8') ^obE9QDcY

depthFrame = self.bridge.imgmsg_to_cv2(depth_data, desired_encoding='passthrough')#"32FC1") ^Rpp9cPLS

if(np.shape(frame)[0:2] != (self.pictureHeight, self.pictureWidth)): ^PZZxi1Kw

raise ValueError('image does not have the right shape. shape(frame): {}, shape parameters:{}'.format(np.shape(frame)[0:2], (self.pictureHeight, self.pictureWidth))) ^EwxXqw8e

# blure a little and convert to HSV color space # 模糊一点，转换成HSV颜色空间 ^YHr2dr8n

#blurred = cv2.GaussianBlur(frame, (11,11), 0) ^iTTE6Sgk

hsv = cv2.cvtColor(frame, cv2.COLOR_RGB2HSV) ^HZdGvmcc

# select all the pixels that are in the range specified by the target ^U5V8hd8O

# 选择目标指定范围内的所有像素 ^d3C6Cj1L

# clean that up a little, the iterations are pretty much arbitrary ^1u8XGBrN

#mask = cv2.erode(org_mask, None, iterations=4) ^FgfmKlRD

# mask = cv2.dilate(mask,None, iterations=3) ^qxX6kksv

kernel0 = np.ones((5,5),np.uint8) ^y0Ks8hlY

hsv_erode = cv2.erode(hsv,kernel0,iterations=1) ^ld9m8Uo6

hsv_dilate = cv2.dilate(hsv_erode,kernel0,iterations=1) ^OchMFhWM

mask_ori = cv2.inRange(hsv_dilate, self.targetUpper, self.targetLower) ^cW3iV2qK

kernel1 = np.ones((3,3),np.uint8) ^hBQfsLWo

mask_erode = cv2.erode(mask_ori,kernel1,iterations=1) ^KUZDIg4A

mask = cv2.dilate(mask_erode,kernel1,iterations=1) ^WarWIhKT

#cv2.imshow("mask", mask) ^HPeKVd3B

cv2.waitKey(3) ^WZeKc5Oh

# find contours of the object 查找对象的轮廓 ^UgysEOhS

contours = cv2.findContours(mask.copy(), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)[-2] ^oUDClVUr

newPos = None #if no contour at all was found the last position will again be set to none ^MutXF5hY

# lets you display the image for debuging. Not in realtime though ^saUCxX7P

# 如果根本没有找到等高线，最后一个位置将再次设置为无。 #允许您显示用于调试的图像。但不是实时的 ^Cd3dn6NW

if self.i < 10: ^tQ9c5Zmi

self.i = self.i +1 ^15Ud7wOb

elif self.i == 10:                #语音识别标志 ^QC0u3WeC

self.publish_flag() ^Qh85t3Wa

self.i = 11 ^CWnwAd3H

if displayImage: ^3kRnFbS8

backConverted = cv2.cvtColor(hsv, cv2.COLOR_HSV2BGR) ^2kc8vgoj

#cv2.imshow('frame', backConverted) ^ZjmA8iDd

#cv2.waitKey(0) ^YyyBYUg9

#print(backConverted) ^AqbVNieH

plt.figure() ^q3OsGkuv

plt.subplot(2,2,1) ^xEUiQtC9

plt.imshow(frame) ^vPwzCrYN

plt.xticks([]),plt.yticks([]) ^SW8AYaku

plt.subplot(2,2,2) ^3fW2Wq9X

plt.imshow(org_mask, cmap='gray', interpolation = 'bicubic') ^P3aqS188

plt.xticks([]),plt.yticks([]) ^OcwKSYwr

plt.subplot(2,2,3) ^sW92v73M

plt.imshow(mask, cmap='gray', interpolation = 'bicubic') ^kc9FOdZC

plt.xticks([]),plt.yticks([]) ^kbXLVYT8

plt.show() ^Tn4tNLI5

rospy.sleep(0.2) ^mHlNs0T5

try: ^eSfAXSRV

# go threw all the contours. starting with the bigest one  抛出了所有的轮廓线。从最大的那个开始 ^9m62gwUO

contour = sorted(contours, key=cv2.contourArea, reverse=True)[0] ^HkSMutnz

# get position of object for this contour 获取此轮廓的对象位置 ^RbgPjcMN

pos = self.analyseContour(contour, depthFrame) ^ke9Axv9N

# if it's the first one we found it will be the fall back for the next scan if we don't find a plausible one ^qMec30Iu

#如果这是我们发现的第一个，如果我们找不到可信的，它将是下一次扫描的后备 ^xo36vWRz

if newPos is None: ^EvXjjOy2

newPos = pos ^mh1TVBIJ

# check if the position is plausible ^KjxwiJqA

#if self.checkPosPlausible(pos): ^VxSsbe89

#检查位置是否合理。 ^SBrQMllc

#if self.checkPosPlausible(位置)： ^RsU5qoAY

self.lastPosition = pos ^PDhCoMON

self.publishPosition(pos) ^Obcv7tmK

return ^5h5IG2b1

#我们没有找到可信的最后一个位置，所以我们只保存了最大的等高线 ^YmLimavy

self.lastPosition = newPos 
#we didn't find a plossible last position, so we just save the biggest contour ^Cd6dfZ2Z

except IndexError: ^HatPqCE4

# and publish warnings ^afrVfUy1

# 打印警告信息 ^I31Bhcy1

rospy.logwarn('no position found') ^OOHp8j5S

posMsg = PositionMsg(0, 0,self.targetDist) ^dzTTVQd3

self.positionPublisher.publish(posMsg) ^bciuhmUF

def publishPosition(self, pos): ^psISPv7X

# calculate the angles from the raw position ^LrsllUlJ

#从原始位置计算角度 ^qIybyhIi

angleX = self.calculateAngleX(pos) ^FTIBhCUx

angleY = self.calculateAngleY(pos) ^Hk1UG4vC

# publish the position (angleX, angleY, distance) ^kMp7oSex

posMsg = PositionMsg(angleX, angleY, pos[1]) ^UxMhSwmH

self.positionPublisher.publish(posMsg) ^WvwjOUbm

def calculateAngleX(self, pos): ^3oxyf0kC

'''calculates the X angle of displacement from straight ahead''' ^CVQmuHju

'''“检查某个位置是否合理，即是否足够接近上一个位置。”''' ^Zahaxr5i

centerX = pos[0][0] ^cDg5AeJT

displacement = 2*centerX/self.pictureWidth-1 ^zJQHXj9h

angle = -1*np.arctan(displacement*self.tanHorizontal) ^FBTELsh2

return angle ^psrxis44

def calculateAngleY(self, pos): ^O50iQU8T

'''calculates the X angle of displacement from straight ahead
从正前方计算位移的X角度''' ^0VNwy7pn

centerY = pos[0][1] ^TH7QVP4x

displacement = 2*centerY/self.pictureHeight-1 ^LrIvaOA1

angle = -1*np.arctan(displacement*self.tanVertical) ^wf3TMgmC

return angle ^NoQMg9dR

def analyseContour(self, contour, depthFrame): ^CdT5njvr

'''Calculates the centers coordinates and distance for a given contour ^Lx5DcCZ7

计算给定等高线的中心坐标和距离 ^Fu55fLcl

Args: ^F8EwNymM

contour (opencv contour): contour of the object ^2FheqWJb

depthFrame (numpy array): the depth image ^Qi3kLS8t

Returns: ^7UHQ3Emp

centerX, centerY (doubles): center coordinates ^dxgi9R9a

averageDistance : distance of the object ^DNH2zEcz

参数： ^GOp0DZu6

轮廓(OpenCV轮廓)：对象的轮廓。 ^6bMUwg2W

Deep thFrame(Numpy Array)：深度图像。 ^HATDxSDT

返回： ^acQpf7D7

中心X、中心Y(双精度)：中心坐标。 ^Mg8MVJEa

AverageDistance：对象的距离 ^5gxYbp3M

''' ^eIpBGeHl

# get a rectangle that completely contains the object ^9TPNfgie

# 获取完全包含该对象的矩形 ^dDivK5py

centerRaw, size, rotation = cv2.minAreaRect(contour) ^fETGFUqr

# get the center of that rounded to ints (so we can index the image) ^JXnZOPTv

center = np.round(centerRaw).astype(int) ^Lp1tKmAR

# find out how far we can go in x/y direction without leaving the object (min of the extension of the bounding rectangle/2 (here 3 for safety)) ^UMdW1FgB

# 找出在不离开对象的情况下，我们可以在x/y方向上走多远(最小边界矩形的延伸长度/2(为安全起见，此处为3)) ^DAu8Z8zu

minSize = int(min(size)/3) ^c2fieSF4

# get all the depth points within this area (that is within the object) ^k45dJOeI

# 获取该区域内（即对象内）的所有深度点 ^PL4WZ3lb

depthObject = depthFrame[(center[1]-minSize):(center[1]+minSize), (center[0]-minSize):(center[0]+minSize)] ^c5VMlyZs

# get the average of all valid points (average to have a more reliable distance measure) ^bFjBfn5c

# 获得所有有效点的平均值（平均值以获得更可靠的距离度量） ^gvwQd3oj

depthArray = depthObject[~np.isnan(depthObject)] ^dMa4lKuW

averageDistance = np.mean(depthArray) ^JbiUIgXa

if(averageDistance>400 or averageDistance<3000): ^eL8Xn8t0

pass ^3324wChq

else: ^fz6q1Qnk

averageDistance=400 ^Nt9rHu50

if len(depthArray) == 0: ^v5UXL46F

rospy.logwarn('empty depth array. all depth values are nan') ^fYtZ1Lgh

return (centerRaw, averageDistance) ^C9se2YvB

# Dynamic parameter configuration ^BfRszbTY

# 动态参数配置 ^334QtlLy

def colorreconfigure(self, config, level): ^M2OimqwM

self.color = config.color ^B7nz3Ask

if self.color== 0:  # ^3ZmoOpHy

HSV_H_MIN =config.HSV_H_MIN ^DsHlUtfV

HSV_S_MIN =config.HSV_S_MIN ^Et5n9gsv

HSV_V_MIN =config.HSV_V_MIN ^5jA0Zmx3

HSV_H_MAX =config.HSV_H_MAX ^3QEraci0

HSV_S_MAX =config.HSV_S_MAX ^xXsa8T2A

HSV_V_MAX =config.HSV_V_MAX ^WyZ6Qdza

self.targetUpper=np.array([HSV_H_MIN,HSV_S_MIN,HSV_V_MIN]) ^YK5c9QeE

self.targetLower=np.array([HSV_H_MAX,HSV_S_MAX,HSV_V_MAX]) ^NCRCk81f

elif self.color == 1:  # ^TSkLAoaz

self.targetUpper=self.col_red_U ^jOrynyvc

self.targetLower=self.col_red_L ^CQui8duh

elif self.color== 2:  # ^A9Io3com

self.targetUpper=self.col_blue_U ^v5PRwAcE

self.targetLower=self.col_blue_L ^YwgtnsM3

elif self.color== 3:  # ^6D5OPvb9

self.targetUpper=self.col_green_U ^etaPgndJ

self.targetLower=self.col_green_L ^EohSe1Nt

elif self.color== 4:  # ^eeEj8Zzv

self.targetUpper=self.col_yellow_U ^QW9L9fXw

self.targetLower=self.col_yellow_L ^0GGaUmfs

return  config ^BaIQfged

if __name__ == '__main__': ^ywc7wLa2

visualfwflagPublisher = rospy.Publisher('/visual_follow_flag', Int8, queue_size =1) ^EmndpXu7

rospy.init_node('visual_tracker',anonymous = False) ^DFBSmNjM

tracker=visualTracker() ^RRc4Bc77

rospy.logwarn('visualTracker init done') ^UKvO1QaZ

try: ^fw6a3me5

rospy.spin() ^FKpnqpFV

except rospy.ROSInterruptException: ^zKcIKwk7

rospy.logwarn('failed') ^tNpmH3zK

1.导入所需库 ^lHV3Ufjx

ROS Python 库，用于创建 ROS 节点并与 ROS 系统交互。 ^glK3a57A

用于同步多个传感器话题。 ^by6ySWP8

OpenCV 库，用于图像处理。 ^SJxQWnjh

用于转换 ROS 图像消息和 OpenCV 图像之间的格式。 ^lIZjLsLJ

用于显示图像（用于调试）。 ^3LSjKgfs

ROS 图像消息类型。 ^6MYT8X34

自定义的消息类型，用于发布物体的位置。 ^El8NrOft

ROS 消息类型，用于发送整数数据。 ^hE3WSPvI

ROS 动态重新配置服务器，用于实时调整参数。 ^PF3zwUaa

2. 初始化和参数 ^uQ9JvJ2N

创建 CvBridge 实例，用于将 ROS 图像消息转换为 OpenCV 图像格式。 ^WfwSkS6f

计数器，用于管理某些动作（如触发标志发布）。 ^iSmxdxsE

动态重新配置服务器，用于实时调整颜色检测参数。 ^yQOicwYd

3. 从 ROS 参数服务器加载参数 ^RtOQ327h

目标颜色的阈值（红色、蓝色、绿色、黄色）从 ROS 参数服务器加载。
这些阈值定义了在 HSV 色彩空间中进行物体检测的范围。 ^1FBpZPhD

4. 相机参数 ^8cpme5Vd

相机的尺寸和视场角被加载。
这些参数用于根据物体在图像中的位置计算角度。 ^jTJ5mLcq

5. 图像和深度数据同步 ^EVy93MGI

为 RGB 图像 和 深度图像 创建订阅者。 ^cI7fNRUJ

使用 ApproximateTimeSynchronizer 同步 RGB 和深度图像帧，
当两帧都可用时，会触发 trackObject 回调函数。 ^73cJ2xTE

6. 目标追踪和位置发布 ^pthEDbIj

检查传入的图像编码是否为 rgb8 格式。
如果不是，抛出异常。这是为了确保图像数据符合预期格式。 ^RiYAdwgu

如果图像编码不是 rgb8，则抛出一个错误并终止函数执行。 ^PwuVLsDb

使用 CvBridge 将 ROS 图像消息 image_data 转换为 OpenCV 格式的图像（RGB格式）。转换后，frame 就是一个包含图像数据的 NumPy 数组。 ^Bu3ucYFX

将 ROS 深度图像消息 depth_data 转换为 NumPy 数组，保留原始数据格式。passthrough 表示不做编码转换，直接传递原始深度值。 ^xtQr5pkj

检查图像的尺寸是否符合预期。如果图像的高度和宽度与 self.pictureHeight 和 self.pictureWidth 不匹配，抛出异常。确保图像大小正确，以便后续处理。 ^JeLC5If6

如果图像尺寸不符合要求，抛出一个 ValueError 异常。 ^iFb6EqPz

使用 OpenCV 的 cv2.cvtColor() 函数将 RGB 图像转换为 HSV 色彩空间。这是因为 HSV 色彩空间更容易分离不同颜色的物体，特别是对于颜色追踪非常有效。 ^uTotF5AE

创建一个 5x5 的卷积核，用于图像腐蚀和膨胀操作。 ^xFCSYdK4

使用腐蚀操作来去除图像中的噪声。腐蚀会把目标物体的边缘向内收缩，
去除不必要的细节。 ^9GFVkTEx

使用膨胀操作来扩展目标物体的边缘，填补腐蚀过程中留下的空洞。 ^f7k2jcVP

根据预先定义的颜色范围 (self.targetUpper, self.targetLower)，
生成一个二值化的掩膜图像。
只有在 HSV 色彩空间中符合目标颜色范围的像素会被保留，其他像素会被设为零。 ^mh4pt0lw

创建一个 3x3 的卷积核，用于进一步的图像处理。 ^5Nar1LpO

对掩膜图像进行腐蚀处理，去除小的噪声。 ^bQgHvoiN

对腐蚀后的掩膜图像进行膨胀处理，恢复物体的形状。 ^HPc8kZQI

7.发布物体位置 ^bb4eMpeh

根据物体位置计算水平方向的角度（X轴）。 ^R1AzDC1E

根据物体位置计算垂直方向的角度（Y轴）。 ^dd75h3Wr

创建一个 PositionMsg 消息，包含计算出的角度和物体的距离（pos[1]）。 ^bT4MQa5u

将位置消息发布到 ROS 话题 ^YpL48L0U

8.轮廓分析和物体距离 ^Y9D2tcZF

使用 OpenCV 的 cv2.minAreaRect() 函数获取轮廓的最小外接矩形。
返回的 centerRaw 是矩形的中心坐标，size 是矩形的宽度和高度，rotation 是矩形的旋转角度。 ^ca4NwF7t

将矩形中心的浮动坐标值四舍五入并转换为整数，以便在图像中进行索引操作。 ^CD210K4W

计算矩形最小边长的三分之一，用于确定计算深度时的区域范围。 ^cq6RyKAi

从深度图像中提取一个矩形区域，区域大小由 minSize 决定，该区域包含物体的中心。 ^uExVIlK1

将深度区域中的有效数据提取出来，排除 NaN 值。 ^UqGGr3XH

计算该区域的平均深度值，即物体的平均距离。 ^CcJZjBEv

如果计算出的距离值超出了合理范围（大于400或小于3000），则不做处理。 ^DPgiHI9D

如果距离值不在合理范围内，则将距离设置为默认值400。 ^WJiQw0b5

9.动态重新配置颜色检测参数 ^TeMOK7BR

通过 ROS 动态重新配置服务器，用户可以在运行时调整颜色检测的参数（如 HSV 范围）。 ^sgunJYU4

10. 初始化 ROS 节点 ^LCP6me5V

创建发布器，发布视觉跟踪标志。 ^yueboU0j

初始化 ROS 节点，并创建 visualTracker 实例。 ^ANMDB7Tn

11. ROS 节点循环 ^II7UmmbU

让节点继续运行，处理传入的数据。 ^YMBd0tP8

在 ROS 中创建一个发布器，话题名为 /object_tracker/current_position，消息类型为 PositionMsg（自定义的包含角度和距离的消息）。 ^twOwqmwO

在控制台以 WARNING 级别打印出当前的 HSV 上界限数组 self.targetUpper，帮助调试时查看阈值是否正确加载。 ^tyYoXh3b

定义一个方法 publish_flag，用于发布“视觉跟随就绪”标志。 ^lSwcRPin

创建一个 Int8 消息实例 ^1HJfhuTT

将标志值设为 1（通常表示启动或激活）。 ^1RE1mQr1

暂停 1 秒，确保订阅方已经准备好接收此标志。 ^UglcJajx

使用在脚本底部全局创建的发布器 visualfwflagPublisher（发布到 /visual_follow_flag）发送这条标志消息。 ^AdCXv0Oc

在控制台以 INFO 级别打印日志，显示发布的标志值（a=1）。 ^BaWsuaHd

在标准输出再打印一串 “1”，可能是用来直观确认函数被调用。 ^x5WDtI9Y

仅在 displayImage=True 时才运行。 ^EJARmsV6

backConverted：将 HSV 图像 hsv 转回 BGR——只是为了显示而已。 ^Ti5DZk5R

用 Matplotlib 打开一个新图窗，分成 2×2 网格的第 1 格，显示原始 RGB 图像 frame，并去掉坐标轴刻度。 ^Qzcc1RNN

第 2 格显示未经过腐蚀/膨胀的原始二值掩膜 org_mask（灰度图）。 ^4B5Hfv0s

第 3 格显示处理后的掩膜 mask，然后 plt.show() 阻塞式地弹出图窗，且用 rospy.sleep(0.2) 保持 0.2 秒以便观察。 ^kUpjURrE

contours 是前面 cv2.findContours(...) 得到的轮廓列表。 ^ouEEz01R

contours 是前面 cv2.findContours(...) 得到的轮廓列表。 ^pS0Kj8Bq

[0]：取面积最大的轮廓，用作目标。 ^ZorAQrHq

调用第 7 点中详解的 analyseContour 函数，计算该轮廓的中心坐标和平均深度，返回 (centerRaw, averageDistance)。 ^yHt4kQ11

newPos 初始为 None，如果第一次找到轮廓，就把它当作“备选”位置存到 newPos。 ^4GswIr3M

无论检查是否通过，直接把当前 pos 存到实例变量 self.lastPosition。 ^tYEh9fOf

调用 publishPosition(pos) 发布这个位置（角度 + 距离）。 ^I2eNvilV

return 跳出 trackObject 方法，不再执行后面的 self.lastPosition = newPos。 ^B7HWftvA

如果 contours 为空、sorted(...)[0] 抛出 IndexError，说明当前帧没检测到目标： ^4Y1skTCu

打印警告； ^QUxryY5x

构造一个默认位置消息 (0, 0, targetDist)； ^mDxwLi3N

发布到 /object_tracker/current_position，通知上层目标丢失或超出范围。 ^GI24ZqvH

python 类的自身引用 ^WUoPiDOY

%%
## Drawing
```compressed-json
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
```
%%