---

excalidraw-plugin: parsed
tags: [excalidraw]

---
==⚠  Switch to EXCALIDRAW VIEW in the MORE OPTIONS menu of this document. ⚠== You can decompress Drawing data with the command palette: 'Decompress current Excalidraw file'. For more info check in plugin settings under 'Saving'


# Excalidraw Data

## Text Elements
#include <iostream> ^T8PwSIV5

#include <fstream> ^FfLVW4Bu

#include <sstream> ^6H7giNg3

#include <algorithm> ^c3atRcmr

#include <dirent.h> ^fZKc6BTw

#include </usr/include/opencv2/highgui/highgui_c.h> ^jl2m4vu3

#include <ros/ros.h> ^s7AJguh9

#include <image_transport/image_transport.h> ^GsShwCMD

#include <cv_bridge/cv_bridge.h> ^0vpNXNd6

#include <sensor_msgs/image_encodings.h> ^l7ssgtyh

#include "geometry_msgs/Twist.h" ^CwMf4Ixw

#include <opencv2/core/core.hpp> ^eU6mtsEh

#include <opencv2/highgui/highgui.hpp> ^hpavjHYn

#include "kcftracker.hpp" ^Ibg9fzx5

static const std::string RGB_WINDOW = "RGB Image window"; ^x93uJYQc

//static const std::string DEPTH_WINDOW = "DEPTH Image window"; ^pzNYsgcs

#define Max_linear_speed 0.6 ^A3HEdlp3

#define Min_linear_speed 0.2 ^L9pu4zST

#define Min_distance 0.7 ^1zzOlJye

#define Max_distance 1.0 ^dSd7AGqA

#define Max_rotation_speed 0.75 ^rR2z0g16

float linear_speed = 0; ^XBy13TNF

float rotation_speed = 0; ^XQmDmGds

float line_distance = 0; ^UyvrMCMR

float angle_distance = 0; ^xfMn39vU

float angle_distance2 = 0; ^IcVCpE9d

float k_linear_speed = (Max_linear_speed - Min_linear_speed) / (Max_distance - Min_distance); ^aaCdjnlL

float h_linear_speed = Min_linear_speed - k_linear_speed * Min_distance; ^3eoG38Y0

float k_rotation_speed = 0.004; ^WpxxsKxn

float h_rotation_speed_left = 1.2; ^sBYVytu9

float h_rotation_speed_right = 1.36; ^BXix2gdr

int ERROR_OFFSET_X_left1 = 100; ^RJuzdBT5

int ERROR_OFFSET_X_left2 = 300; ^ZeC5v6vh

int ERROR_OFFSET_X_right1 = 340; ^8NQYO5B2

int ERROR_OFFSET_X_right2 = 540; ^dSsdvZ5N

cv::Mat rgbimage; ^McimGp8B

cv::Mat depthimage; ^TN18peXa

cv::Rect selectRect; ^vq9pSBUv

cv::Point origin; ^QRsQgs9l

cv::Rect result; ^XuDIrJBW

bool select_flag = false; ^69T4qNfo

bool bRenewROI = false; ^QcjsMB0U

bool bBeginKCF = false; ^ordoH5eI

bool enable_get_depth = false; ^aeKGzYVJ

bool HOG = true; ^CG74IdSp

bool FIXEDWINDOW = false; ^FIxdVa34

bool MULTISCALE = true; ^sOuqEqSz

bool SILENT = true; ^4O9ZIPIw

bool LAB = false; ^ygy5897l

KCFTracker tracker(HOG, FIXEDWINDOW, MULTISCALE, LAB); ^XkqobJFI

float dist_val[5] ; ^FAyY8WEw

void onMouse(int event, int x, int y, int, void*) ^7DJ4qCoG

{ ^wJTqxH1x

if (select_flag) ^rx9WEzCf

{ ^VtypOfgT

selectRect.x = MIN(origin.x, x); ^9srA5KfT

selectRect.y = MIN(origin.y, y); ^qJuCEQ0B

selectRect.width = abs(x - origin.x); ^AaeCllCn

selectRect.height = abs(y - origin.y); ^p39J5I6O

selectRect &= cv::Rect(0, 0, rgbimage.cols, rgbimage.rows); ^dRQHiZj3

} ^KeUAR6Q5

if (event == CV_EVENT_LBUTTONDOWN) ^7CgN6Dut

{ ^ZJi09m88

bBeginKCF = false; ^JeuNtEEu

select_flag = true; ^kOQEcmSv

origin = cv::Point(x, y); ^jxwdD93n

selectRect = cv::Rect(x, y, 0, 0); ^M172cwxS

} ^ylvoin74

else if (event == CV_EVENT_LBUTTONUP) ^8PU4FTPI

{ ^DIeb7KIz

select_flag = false; ^Snub5vWy

bRenewROI = true; ^YD8uKIxU

} ^GZSBWerJ

} ^anqWx9ya

class ImageConverter ^wsSxNOMP

{ ^WJH6QThr

ros::NodeHandle nh_; ^QMaasVCo

image_transport::ImageTransport it_; ^CivRjggI

image_transport::Subscriber image_sub_; ^3tYib9z9

image_transport::Subscriber depth_sub_; ^8ApWqG2G

std::string rgb_topic; ^ggIzlxJh

std::string depth_topic; ^pHStGFPM

public: ^klubWiFQ

ros::Publisher pub; ^1SR02DRS

ImageConverter() ^0UIe1cc7

: it_(nh_) ^e6NNdMxk

{ ^dSLQIqDf

nh_.param<std::string>("/kcf_tracker/rgb_topic", rgb_topic,"/camera/rgb/image_raw"); ^vubBkbbT

nh_.param<std::string>("/kcf_tracker/depth_topic", depth_topic,"/camera/depth/image"); ^Tgx8UlCR

// Subscrive to input video feed and publish output video feed ^mOoDMy53

/*image_sub_ = it_.subscribe("/camera/rgb/image_raw", 1, ^qrx4I2wT

&ImageConverter::imageCb, this); ^sNhKLcbt

depth_sub_ = it_.subscribe("/camera/depth/image", 1, ^UPKS8nIW

&ImageConverter::depthCb, this);*/ ^Ee5smw8i

image_sub_ = it_.subscribe(rgb_topic, 1, ^N9OSchL7

&ImageConverter::imageCb, this); ^AK4HPJSZ

depth_sub_ = it_.subscribe(depth_topic, 1, ^ArFWdN37

&ImageConverter::depthCb, this); ^oA3soDOA

pub = nh_.advertise<geometry_msgs::Twist>("kcf/track", 1000); ^u0M7EIoG

//pub = nh_.advertise<geometry_msgs::Twist>("position/distance", 1000); ^auDXiguP

cv::namedWindow(RGB_WINDOW); ^38X4Qt2E

//cv::namedWindow(DEPTH_WINDOW); ^M2kBmml8

} ^dqQXQuGB

~ImageConverter() ^HegwUB2U

{ ^DLY1IXi7

cv::destroyWindow(RGB_WINDOW); ^5gIHqPI0

//cv::destroyWindow(DEPTH_WINDOW); ^pxVwRwQb

} ^TcbKaqvl

void imageCb(const sensor_msgs::ImageConstPtr& msg) ^TKvoX8Ff

{ ^2eJCRpIC

cv_bridge::CvImagePtr cv_ptr; ^cJw3KQQy

try ^U2PsqQvE

{ ^znB6MK1J

cv_ptr = cv_bridge::toCvCopy(msg, sensor_msgs::image_encodings::BGR8); ^ZWJIPPRU

} ^E05SHIYV

catch (cv_bridge::Exception& e) ^ZbtHiYFd

{ ^4MqkrKjE

ROS_ERROR("cv_bridge exception: %s", e.what()); ^JiS1PIGR

return; ^ZtsPg5ph

} ^pnFsaUCi

cv_ptr->image.copyTo(rgbimage); ^s18nM8Ws

cv::setMouseCallback(RGB_WINDOW, onMouse, 0); ^UZ2Z8wIF

if(bRenewROI) ^GyZG73aX

{ ^UcDJUW6X

// if (selectRect.width <= 0 || selectRect.height <= 0) ^ScrO2fm2

// { ^c9lwGiF0

//     bRenewROI = false; ^v4vDM40t

//     //continue; ^H6Vf5EGa

// } ^8ZPaGyHb

tracker.init(selectRect, rgbimage); ^DFCLH1Fz

bBeginKCF = true; ^c8lTpymH

bRenewROI = false; ^VnKPSs6p

enable_get_depth = false; ^HHe3b3Zi

} ^vSe6ZnAD

if(bBeginKCF) ^FWcse7fA

{ ^dtjVGyGg

result = tracker.update(rgbimage); ^XtJdWBBD

cv::rectangle(rgbimage, result, cv::Scalar( 0, 255, 255 ), 1, 8 ); ^CK1RNLZM

enable_get_depth = true; ^sYYxQUNF

} ^LQA5CSUN

else ^1FIF3SvJ

cv::rectangle(rgbimage, selectRect, cv::Scalar(255, 0, 0), 2, 8, 0); ^7RWprDwv

cv::imshow(RGB_WINDOW, rgbimage); ^pMcQ1LMf

cv::waitKey(1); ^adsqpC0O

} ^SPcCJ8kU

void depthCb(const sensor_msgs::ImageConstPtr& msg) ^kHhW8no5

{ ^7odk1qK5

cv_bridge::CvImagePtr cv_ptr; ^rG0OtDtU

try ^AYq2Pbm4

{ ^t12EICUz

cv_ptr = cv_bridge::toCvCopy(msg,sensor_msgs::image_encodings::TYPE_32FC1); ^SK5XNBGA

cv_ptr->image.copyTo(depthimage); ^JFd8HT7w

} ^mDAvuGMr

catch (cv_bridge::Exception& e) ^Ni5yjwAD

{ ^Ph09RtiS

ROS_ERROR("Could not convert from '%s' to 'TYPE_32FC1'.", msg->encoding.c_str()); ^vPOQAhjI

} ^oHv9rxez

if(enable_get_depth) ^exrnpJI8

{ ^hRSiDQGZ

dist_val[0] = depthimage.at<float>(result.y+result.height/3 , result.x+result.width/3) ; ^sI0k6Evl

dist_val[1] = depthimage.at<float>(result.y+result.height/3 , result.x+2*result.width/3) ; ^UXKkULNx

dist_val[2] = depthimage.at<float>(result.y+2*result.height/3 , result.x+result.width/3) ; ^Ho1ZmltX

dist_val[3] = depthimage.at<float>(result.y+2*result.height/3 , result.x+2*result.width/3) ; ^HJqspvwA

dist_val[4] = depthimage.at<float>(result.y+result.height/2 , result.x+result.width/2) ; ^w0rBdfA0

float distance = 0; ^FcyJOOpH

int num_depth_points = 5; ^ho9G2hm5

for(int i = 0; i < 5; i++) ^lzV5ntRn

{ ^NIYSMIZC

if(dist_val[i] > 0.4 && dist_val[i] < 10.0) ^nsKmYueB

distance += dist_val[i]; ^72bx3RZh

else ^aklCTFuo

num_depth_points--; ^rj5bFDV7

} ^KcMap5zE

distance /= num_depth_points; ^THsw8JOY

if(distance>0.6)  distance = distance; ^1uOSMyrg

else            distance = 0.6; ^6AmqPaFT

line_distance = distance; ^kvgfFiRf

//calculate linear speed ^UWxHZo0g

/* ^ZUEHxusw

if(distance > Min_distance) ^u3iHYQG6

linear_speed = distance * k_linear_speed + h_linear_speed; ^eVRGiO1e

else ^UsLpK63u

linear_speed = 0; ^gR8Ug0il

if(distance > Max_distance) ^AHq1IeuF

linear_speed = 0.6; ^wEkUd4cU

if(distance < Min_distance) ^0RZutwMr

linear_speed = -0.6; ^9ZsgOR5W

if(linear_speed > Max_linear_speed) ^oLxQpTyO

linear_speed = Max_linear_speed; ^xxqBVVN1

*/ ^beUPsifK

//calculate rotation speed ^hshTx7CO

int center_x = result.x + result.width/2; ^sy3cAVB6

int center_y = result.y + result.height/2; ^xM8UfHUv

angle_distance2=center_y; ^vk52rNcB

angle_distance = center_x; ^Vznam3a2

引入了多种标准库和ROS、OpenCV相关库： ^r3cbNHYE

用于处理文件、字符串、目录等基本功能。 ^eNeHAVXB

OpenCV核心库和图形用户界面相关功能。 ^pcRJ3S4p

ROS基本接口，允许创建ROS节点。 ^lveYFwdZ

用于订阅和发布ROS图像消息。 ^jIdQHgwM

用于将ROS图像消息和OpenCV图像互相转换。 ^fccQyBpq

用于发布ROS消息，控制小车的线速度和角速度。 ^8ocLG85E

自定义的KCF跟踪器类的头文件。 ^uVe4leXi

定义了小车的最大最小线速度、最大最小目标距离和
最大旋转速度等控制参数。 ^ux5P3EJv

当前线速度和角速度的值。 ^dXYpnV98

用于保存计算得到的目标距离和角度偏差。 ^xFSPmeLU

线速度的计算系数，确保目标距离和线速度的比例。 ^Z5z7EGr9

旋转速度的计算系数。 ^st9b5gkt

这些常量用于定义目标框在图像中的位置范围，以便控制小车的旋转。 ^j7vWTXCs

用于存储RGB图像和深度图像。 ^SNYMKWIz

创建KCF跟踪器对象，初始化时传入一些参数。 ^LaQ2rm4M

标志是否正在选择目标区域。 ^q1d3baAa

标志是否需要更新ROI并重新初始化KCF跟踪器。 ^i4JY57G9

标志KCF跟踪是否已开始。 ^tPT9NwTS

标志是否启用深度图像获取。 ^VBJ75PXf

KCF跟踪器相关变量： ^DCDGf3EU

常量定义： ^7DskRjNN

定义RGB图像显示窗口的名称，使用OpenCV进行显示。 ^a3wxIvBk

变量初始化 ^PxxXsMbz

标志变量： ^JdylYwyW

这些布尔值控制KCF跟踪器的参数，
如是否使用HOG特征、是否固定窗口、
是否使用多尺度、是否静默运行等。 ^F78CETVu

创建KCF跟踪器对象： ^1rITrvoL

鼠标回调函数 (onMouse)： ^YVvNfMhT

图像转换类 ImageConverter： ^cJQZ9n27

类用于订阅图像消息并处理图像。 ^zghX39dJ

构造函数： ^FCaDKkXh

图像回调函数： ^dah60xFo

深度图像回调函数： ^dwaTrm9u

/* ^DDp7FPAO

if(center_x < ERROR_OFFSET_X_left1) ^NrxLwvBO

rotation_speed =  Max_rotation_speed; ^CU6gUh4m

else if(center_x > ERROR_OFFSET_X_left1 && center_x < ERROR_OFFSET_X_left2) ^IA3Sfgqk

rotation_speed = -k_rotation_speed * center_x + h_rotation_speed_left; ^jhms6vCj

else if(center_x > ERROR_OFFSET_X_right1 && center_x < ERROR_OFFSET_X_right2) ^6G7ntAff

rotation_speed = -k_rotation_speed * center_x + h_rotation_speed_right; ^F6xmAz3c

else if(center_x > ERROR_OFFSET_X_right2) ^nTRtno8r

rotation_speed = -Max_rotation_speed; ^GggNZ213

else ^3JMkgX3V

rotation_speed = 0; ^A9QsbiKi

*/ ^e1wZvvbH

std::cout <<  "linear_speed = " << linear_speed << "  rotation_speed = " << rotation_speed << std::endl; ^Cy4O02D6

// std::cout <<  dist_val[0]  << " / " <<  dist_val[1] << " / " << dist_val[2] << " / " << dist_val[3] <<  " / " << dist_val[4] << std::endl; ^PAMguBOC

// std::cout <<  "distance = " << distance << std::endl; ^XqIi2ygk

} ^ERyT3Dqr

//cv::imshow(DEPTH_WINDOW, depthimage); ^vVw2V0WJ

cv::waitKey(1); ^Ge7K4nIT

} ^tsW7xgiJ

}; ^LuS5hxwy

int main(int argc, char** argv) ^c6qkl1Xu

{ ^R5806OPr

ros::init(argc, argv, "kcf_tracker"); ^aaEYhesg

ImageConverter ic; ^kU91oMvR

/* ^8pGiaSxL

private_nh.param<float>("Max_linear_speed", Max_linear_speed, 1.0); //固定串口 ^G9X6AKpZ

private_nh.param<float>("Min_linear_speed", Min_linear_speed, 0.2); //和下位机底层波特率115200 不建议更高的波特率了 ^5ACGoocg

private_nh.param<float>("Min_distance", Min_distance, 0.4);//平滑控制指令 ^tyvcOkzu

private_nh.param<float>("Max_distance", Max_distance, 0.8);//ID ^soHXERBb

private_nh.param<float>("Max_rotation_speed", Max_rotation_speed, 0.7);//ID ^ePiIXE3h

*/ ^q79QqXeK

while(ros::ok()) ^Pz7B7ZBk

{ ^gcDs9tGW

ros::spinOnce(); ^u9UXhahk

geometry_msgs::Twist twist; ^lLLxChcF

twist.linear.x = line_distance; ^QO5M0Dl7

twist.linear.y = 0; ^QmQyRUxD

twist.linear.z = 0; ^UKXy70iT

twist.angular.x = 0; ^DUAFvra5

twist.angular.y = angle_distance2; ^921jIsSb

twist.angular.z = angle_distance; ^GjfN0kz6

ic.pub.publish(twist); ^DFfhBEoR

if (cvWaitKey(33) == 'q') ^zXp3HE5R

break; ^ND7NxVBj

} ^uuCKgJqc

return 0; ^66Z0RClJ

} ^9zLsbRvF

定义一个名为 imageCb 的成员函数，用于接收 ROS 发布的 RGB 图像消息。参数 msg 是一个指向常量 sensor_msgs::Image 消息的智能指针,包含了图像的所有数据（如深度值、图像编码类型等）。 ^7X0f3KPn

声明一个 CvImagePtr 智能指针，用于后续将 ROS 图像消息转换成 OpenCV 格式。 ^IuFkg6Ep

开始一个 try 块，用于捕获图像转换过程中可能抛出的异常。 ^rexrjLdI

调用 cv_bridge::toCvCopy，将 ROS 的 msg （通常是 mono8、rgb8 等编码）转换为 OpenCV 的 BGR8（3 通道）格式，并将结果保存在 cv_ptr 中。 ^8Tz5FrDA

如果转换过程中出现异常（如编码不匹配），则进入这个 catch 分支。 ^27Ei1KAK

使用 ROS 日志宏打印错误信息，将异常描述 e.what() 输出到控制台。 ^QHymwbYo

由于图像无法正确转换，直接返回，不做后续处理。 ^k2Bxxfv2

将 cv_ptr 中的 OpenCV 图像复制到全局变量 rgbimage，以便后续处理。 ^SLVPoX36

为名为 RGB_WINDOW（“RGB Image window”）的 OpenCV 窗口注册鼠标回调函数 onMouse。用户可以用鼠标在窗口中拖拽选定目标区域。 ^NiKUGPB2

如果标志 bRenewROI 为 true，表示用户刚刚完成了新的 ROI（感兴趣区域）选择，需要重新初始化 KCF 跟踪器。 ^kAJvEBFF

用用户选择的矩形 selectRect 和当前帧 rgbimage，初始化 KCF 跟踪器。 ^jdlptRSQ

将跟踪开始标志 bBeginKCF 置为 true，接下来正式进入跟踪模式。 ^pPqz6LQ1

重置 ROI 更新标志，避免重复初始化。 ^iTvvYXvi

enable_get_depth = false; ^qlyALLWa

如果跟踪已开始（bBeginKCF == true），则执行跟踪更新。 ^yKp4DQxr

调用 KCF 跟踪器的 update 函数，用当前帧 rgbimage 更新目标位置，并将新的矩形存入 result。 ^XgcPQHk6

在 rgbimage 上用黄色（BGR=(0,255,255)）、线宽 1、线型 8 绘制当前跟踪的目标矩形 result。 ^ZHfs6amH

设置深度取值标志，允许下一步在深度回调中获取目标的深度信息。 ^2zj800JC

如果跟踪尚未开始（bBeginKCF == false），则执行 else 分支。 ^heryZaDQ

在图像上用蓝色（BGR=(255,0,0)）、线宽 2 绘制用户当前拖拽的选区 selectRect，以便用户可视化调整。 ^1CpSmUTb

将绘制好矩形的 rgbimage 显示到窗口 RGB_WINDOW 上。 ^mkz9vVJw

调用 OpenCV 的事件轮询，每帧等待 1 毫秒，以便窗口能正常刷新并响应键鼠事件。 ^V0zWc9p4

定义名为 depthCb 的成员函数，用于接收 ROS 发布的深度图像消息。 ^WTQxSa2U

声明一个 CvImagePtr 智能指针，用于图像格式转换。 ^44OfX6X0

开始 try 块捕获转换异常。 ^oyfHCOkc

将 ROS 深度消息 msg 转换为 OpenCV 的 32 位浮点单通道格式（每个像素表示实际深度，单位通常为米）。 ^M1QBXti2

将转换后的深度图复制到全局变量 depthimage 中，供后续深度读取使用。 ^04TFLbLC

如果转换失败，进入 catch 分支。 ^VoAevxzk

打印错误日志，指出原始编码无法转换为浮点格式。 ^3wUbJzJ4

仅当上一帧跟踪已完成并设置 enable_get_depth == true 时，才读取深度信息。 ^1fuzCJ9F

分别在目标矩形 result 内取五个关键点的深度值
（左上、右上、左下、右下、中心五点），存入数组 dist_val。 ^KGUUGCSX

定义变量 distance 用于累加有效深度值，num_depth_points 记录有效深度点数量。 ^aTWiFOuw

遍历五个深度值：若某点深度在合理范围（0.4–10.0 米）内则累加，否则说明该点无效，减少计数。 ^CrC3jZiy

将累计深度除以有效点数，得到目标的平均深度。 ^ig2JlUPm

确保最小深度阈值为 0.6 米（防止过近导致速度计算异常）。 ^G2yjadNV

将计算出的深度值保存到全局变量 line_distance，用于后续线速度控制。 ^lP1oLju4

计算目标矩形中心在图像坐标系中的 x、y 值。 ^gd4GuBBJ

将中心坐标分别保存到 angle_distance2（映射为角速度的某一分量）和 angle_distance（映射为另一分量）。 ^AoiFsvxj

在控制台打印当前计算的线速度和角速度，便于调试观察。 ^J9wHjUl2

if (enable_get_depth) 块结束。 ^bplOY2Cl

为了配合 OpenCV GUI 的事件处理，即使不显示深度图，也要调用 waitKey，否则图像窗口和鼠标事件可能失效。 ^d3A1l9Lx

初始化ROS节点
创建ImageConverter对象。 ^pwUhPNVB

在循环中获取跟踪结果，并通过geometry_msgs::Twist发布控制信号，控制小车运动。 ^8dSGbBBA

event：鼠标事件类型（如按下、释放、移动等）。 ^5kwg3oO5

x 和 y：鼠标事件发生时的坐标。 ^fBAsbew7

int：未使用的参数，OpenCV要求的标准函数参数。 ^XTI9BgYV

void*：未使用的参数，OpenCV要求的标准函数参数。 ^MEOIY7sr

如果 select_flag 为 true，表示用户正在拖动鼠标选择目标区域。select_flag 在鼠标按下时被设置为 true，鼠标释放时为 false。 ^5Ty1AO3N

计算当前选择框的左上角的 x 坐标。origin.x 是鼠标按下时的起始坐标，x 是当前鼠标位置。使用 MIN 确保选择框的左上角不会超出图像的边界。 ^zgmLd680

selectRect.y = MIN(origin.y, y);：计算当前选择框的左上角的 y 坐标，原理同上。 ^6xhfXrX4

计算选择框的宽度，abs 确保宽度为正值。 ^8jhxtoix

计算选择框的高度，abs 确保高度为正值。 ^7lEUnAgk

确保选择框的区域不会超出图像的边界。如果选择框的位置或大小超出了图像的范围，就将其调整回有效区域。 ^vd9qFLlD

如果鼠标左键按下时触发此事件（OpenCV定义的左键按下事件 CV_EVENT_LBUTTONDOWN）。 ^A0ESZKBM

当鼠标左键按下时，表示开始选择新的目标区域，因此将 KCF 跟踪的状态标记为 false，即停止当前的跟踪。 ^NKRhthTj

设置 select_flag 为 true，表示当前正在选择目标区域。 ^kK2AkE3f

记录鼠标按下时的坐标，作为选择框的起始点。 ^zH50KQk7

初始化一个矩形框 selectRect，起始位置为鼠标按下时的坐标，宽度和高度为 0，表示开始创建矩形框。 ^asOFmGHn

如果鼠标左键释放时触发此事件（OpenCV定义的左键释放事件 CV_EVENT_LBUTTONUP）。 ^yelVpU7b

设置 select_flag 为 false，表示目标区域的选择结束。 ^2V50qTog

将 bRenewROI 设置为 true，表示需要更新感兴趣区域（ROI），并且会在后续的图像回调函数中重新初始化 KCF 跟踪器。 ^Mc47mXvN

在 OpenCV 中，cv::Mat 类用于存储图像矩阵，可以容纳图像的像素值、通道等信息。在这里，它用于保存 RGB 图像的像素数据。 ^fmQekQs2

cv::Rect 是 OpenCV 中的矩形类，用来表示矩形区域。 ^FtvweYru

矩形由四个参数描述：x 和 y 表示矩形的左上角坐标，width 表示矩形的宽度，height 表示矩形的高度。 ^7LaVGWPL

cv::Point 是 OpenCV 中表示点的类，它包含两个整数：x 和 y，分别表示点的横纵坐标。 ^6bisJ7rv

result 用来存储 KCF 跟踪器的输出矩形区域，即目标在图像中跟踪后的位置和大小。 ^9yYriSc5

用于存储 RGB 图像 ROS 话题名称，允许通过参数动态设置。 ^maHnv4i3

用于存储深度图像的 ROS 话题名称，允许通过参数动态设置。 ^gz5OIFse

ImageTransport 是 ROS 中用于图像传输的类，可以高效地处理图像的发布和订阅。 ^zESIsfxv

ROS 的节点句柄，用于管理节点的操作，例如参数读取、发布者和订阅者的创建等。 ^O8svkZur

图像消息的订阅者，用于订阅 RGB 图像。 ^1T4BPEWy

图像消息的订阅者，用于订阅深度图像。 ^2ttsuotT

构造函数，类对象创建时调用。 ^99UfcFHm

初始化 it_（ImageTransport 对象）时，传入 nh_（NodeHandle 对象），使 it_ 能够访问当前节点。 ^FsGUHgvp

读取 ROS 参数服务器上的话题名称配置，"/kcf_tracker/rgb_topic" 和 "/kcf_tracker/depth_topic" 分别表示用于接收 RGB 图像和深度图像的 ROS 话题。如果参数不存在，则使用默认值 "/camera/rgb/image_raw" 和 "/camera/depth/image"。 ^468nF5sU

image_sub_ = it_.subscribe(...)：创建一个图像订阅者 image_sub_，订阅指定的 RGB 图像话题，并在收到图像消息时调用 ImageConverter::imageCb 函数进行处理。1 表示消息队列的大小，this 表示当前类对象。 ^1PhAjBgo

depth_sub_ = it_.subscribe(...)：类似地，创建一个深度图像订阅者 depth_sub_，订阅深度图像话题，并在收到消息时调用 ImageConverter::depthCb 进行处理。 ^MNyJPAAa

pub = nh_.advertise<geometry_msgs::Twist>("kcf/track", 1000);：创建一个发布者 pub，用于发布 geometry_msgs::Twist 消息类型的控制命令，话题名为 "kcf/track"，队列大小为 1000。 ^1i1tm7It

使用 OpenCV 创建一个名为 RGB_WINDOW 的窗口，用于显示 RGB 图像。窗口的名称在前面定义为 "RGB Image window"。 ^eEJg2MJk

析构函数，在 ImageConverter 对象销毁时调用，用于清理资源。 ^Cf9Io3eV

销毁创建的 OpenCV 图像窗口 RGB_WINDOW，释放相关资源。 ^nu2LT8cO

当 depth_sub_ 订阅的深度图像话题（例如 "/camera/depth/image"）收到新的消息时，ROS 自动将该消息作为参数传递给回调函数 depthCb。 ^jKB5kNqB

this：表示回调函数中的 this 指针，它指向当前 ImageConverter 类的实例。 ^I3sB0EkN

opencv中的setmousecallback类 ^b2oqOA1n

angle_distance 会出现负值，并且它表示 目标中心偏离图像中心的水平距离。当目标在图像的左侧时，angle_distance 为负；当目标在右侧时，angle_distance 为正。这个值会影响后续的角速度计算，用于调整机器人的旋转方向。 ^IIFqsjTW

%%
## Drawing
```compressed-json
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

APxCs8KyY3Sisi/SZuytYi/iA+MQcWKy5LIUspKzlLNSs9KzrfS0s3Sz9LJKs1ld8rPKs+VCdeSKs2yy6hLKspyzZrIN5Vyys9y8s3yz/LMUQpqyWrMis6KyQDLIrcAAX4H/gR+guQFxgbC1CgGgAbKBMgHVgfNBNgAYAQgB/XCsgL6MtlkDzN6yhgFiQEQBF4GLUjIAuQEkRbIyDWC+s4Uz+gH0AZ6zPKKN4T6zUMGBsjIAbpXKVSGzvrJBsv6z

Fnnhs6Gz9ACRspiyPrIdlKGzsgB+s/QBJIB86FGycbJBsinkXLkJsqABcbLggUikybIpshKdAbOxs8myQbIkQE2jU6WpsxGyirX1MwIzO6FZsjIBuwH8M4S1ObNNMxAyYqCBsomzlLV4tWTVWqAagD6yGOiRAfAB7TEpaWUwxfEeMJbEHF0jAVtpZbPvMAnF9xUusowAgPAQwAUytYAIABSBzgGbEbmy8bLBjbZ4PrOJAEgBAeHWwA4wbbP6Abq0

y4Hts4gAJIivAXmyTPGCAAEwXbLLwfggrICRAR1AwMlwAfLFKiAqIDYAw7Ni7NoBVymkgAmdZ4CDs/LFwZKuYX9JrW0jsnSJTbKxsxeB0bIQACnkZtDfqa4onXCvAHHoLrK1Mgu8aYAasC8BCAG6tBqxB9iJ6dqZnWAcUU2y7ADcwRJwRCBtgAbE3ZI9shzxvbIJAYiBGADB+JEBi7OBGWWJEEHMQGHAJIH0ACWzIIAJMqjAILXQgX2x4pn7smuw

xrXCQcAAsSDPicIBsLUwwHSAgAA=
```
%%