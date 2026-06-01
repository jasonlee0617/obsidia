### 第 1 类：硬件接口类（C++，核心逻辑）
#### 1.fairino_hardware_interface.hpp—— 硬件接口的“设计说明书”
👉 **“接口的设计蓝图”**
- 定义类继承关系
- 定义成员变量（状态、命令、机器人对象、参数等）
- 声明 ros2_control 生命周期函数
##### 1️⃣ 类的整体结构
```
class FairinoHardwareInterface
  : public hardware_interface::SystemInterface
```
👉 这句话非常重要：
- **所有 ros2_control 硬件接口**
- **必须继承 `SystemInterface`**
这就是 controller_manager 能“统一调度所有机器人”的根本原因。
##### 2️⃣ 生命周期函数声明（骨架）
```
CallbackReturn on_init(const hardware_interface::HardwareInfo & info) override;
CallbackReturn on_activate(const rclcpp_lifecycle::State &) override;
CallbackReturn on_deactivate(const rclcpp_lifecycle::State &) override;

std::vector<StateInterface> export_state_interfaces() override;
std::vector<CommandInterface> export_command_interfaces() override;

hardware_interface::return_type read(
  const rclcpp::Time &, const rclcpp::Duration &) override;

hardware_interface::return_type write(
  const rclcpp::Time &, const rclcpp::Duration &) override;

```
👉 **这 6 个函数 = ros2_control 硬件接口的“固定接口协议”**
controller_manager 不管你是什么机器人，只认这几个函数
##### 3️⃣ 成员变量：这是“真实的数据通道”
###### (1）关节命令 / 状态数组
```
double _jnt_position_command[6];
double _jnt_position_state[6];
```
👉 **这是 ros2_control 与真实机器人之间的“共享内存”**
- 控制器写 `_jnt_position_command`
- 你在 `write()` 里读它 → 发给机器人
- 你在 `read()` 里写 `_jnt_position_state`
- 控制器读它 → 知道真实位置

📌 **这是 ros2_control 最核心的思想**
> 控制器和硬件接口  
> **通过指针共享数据，而不是函数调用**
###### （2）机器人 SDK 对象
```
std::unique_ptr<FRRobot> _ptr_robot;
```
👉 这是你和真实机械臂通信的“钥匙”
- 在 `on_activate()` 创建
- 在 `on_deactivate()` 释放
- 在 `read/write()` 里调用 SDK API
###### （3）控制模式
```
int _control_mode;
```
- 0：位置控制（ServoJ）
- 1：力矩控制（预留）
- 2：速度控制（预留）
###### （4）控制器 IP 等参数
```
std::string _controller_ip;
```

#### 2.fairino_hardware_interface.cpp—— 硬件接口的“行为说明书”
👉 **“接口的真实实现”**
- 实现 read / write / on_activate / on_deactivate
- 调用真实机器人 SDK
- 完成 ros2_control ↔ 真机 的数据桥接
##### 1️⃣ 头文件与命名空间
###### 1.1 include
文件开头 include 了自己的接口头：
- `#include "fairino_hardware/fairino_hardware_interface.hpp"`
这表示 `.cpp` 是对 `.hpp` 中 `FairinoHardwareInterface` 类的实现。
###### 1.2 namespace
- `namespace fairino_hardware { ... }`
让类名变成：`fairino_hardware::FairinoHardwareInterface`

##### 2️⃣ on_init() —— “配置检查关卡”：URDF 写错就直接拒绝启动
###### 2.1 先调用父类 on_init
```
if (hardware_interface::SystemInterface::on_init(sysinfo) != SUCCESS) return ERROR;
info_ = sysinfo;
```
- `SystemInterface::on_init()` 是 ros2_control 基类的初始化
- `info_ = sysinfo` 把 URDF 解析得到的硬件信息存下来，后面 export interfaces 会用到
###### 2.2 遍历每个 joint，检查 command 接口
代码要求：
- **每个关节 command interface 必须只有 1 个**
- 而且这个接口必须叫 **position**（`HW_IF_POSITION`）
这意味着：  
✅ 你的硬件插件当前只支持 **位置控制（position command）**。  
如果你在 URDF 里声明了 velocity/effort，或声明了多个 command interface，这里直接 `FATAL` 并返回 ERROR。
###### 2.3 检查 state 接口（同样只允许 position）
- state interface 也要求只有 1 个
- 也必须叫 position
`on_init()` 是“你支持什么接口”的**硬性声明**。  
URDF 写错，宁可在这里失败，也不要让机器人带着错误配置跑起来。

##### 3️⃣ export_state_interfaces() —— “把状态内存地址交给 ros2_control”
这是 ros2_control 最核心的机制之一：  
**控制器并不通过函数去读取状态，而是通过你导出的 `double*` 指针直接读内存。**
###### 3.1 创建返回容器
std::vector<hardware_interface::StateInterface> state_interfaces;
###### 3.2 遍历 joints，把 position state 导出
对每个关节：
```
StateInterface(joint_name, HW_IF_POSITION, &_jnt_position_state[i])
```
- “名为 joint_name 的关节”
- “它有一个名为 position 的状态接口”
- “它的值就放在 `_jnt_position_state[i]` 这块内存里”
###### 3.3 return
return state_interfaces;
##### 4️⃣ export_command_interfaces() —— “把命令内存地址交给 ros2_control”
与 state 相对：  
**控制器通过 `double*` 指针写命令**，硬件接口在 `write()` 中读取并发送给机器人。
###### 4.1 创建命令接口数组
std::vector<hardware_interface::CommandInterface> command_interfaces;
###### 4.2 遍历 joints，导出 position command
```
CommandInterface(joint_name, HW_IF_POSITION, &_jnt_position_command[i])
```
当 joint_trajectory_controller（或别的控制器）要写“目标关节角”，写的就是 `_jnt_position_command[i]`。

##### 5️⃣ on_activate() —— “上线阶段”：连接机器人 + 初始化 + 安全对齐
**on_activate 的作用**：硬件从 inactive 切到 active 时调用。  
你必须在这里把硬件通信准备好，并把系统状态调到安全可控。
###### 5.1 打日志 + 创建机器人对象
- `Starting .please wait.`
- `_ptr_robot = std::make_unique<FRRobot>();` 创建 SDK 对象
###### 5.2 初始化变量（当前写死 6 轴）
对 6 个关节，把 command/state（position/velocity/torque）全部清零：
###### 5.3 设置控制模式为位置控制
`_control_mode = 0;`，注释说明 0=位置，1=扭矩，2=速度
###### 5.4 建立 RPC 连接（关键）
- `returncode = _ptr_robot->RPC(_controller_ip.c_str());`
- `sleep_for(200ms)` 等待连接建立
- 失败就 ERROR，成功就继续
###### 5.5 读取当前真实关节角，并做“command = state 对齐”
这一段是整份代码里最“安全关键”的部分。
- 先读当前关节角（单位：度）  
    `GetActualJointPosDegree(0, &jntpos)`
- 代码注释明确写出原因：  
    读到反馈位置后同步到指令位置以维持当前状态；读失败就不能激活，否则初始指令可能偏差导致事故
- 成功则把反馈角度转换为弧度写入 `_jnt_position_command[j]`  
    `deg/180*pi`
- 打印“初始指令位置”，并返回 SUCCESS
- 失败则报错并返回 ERROR
==这一步就是防止“控制器刚启动时命令是 0，机器人突然冲向 0 位”的经典保护策略。==
##### 6️⃣ on_deactivate() —— “下线阶段”：停机 + 断开连接 + 释放资源
- 打印 stopping 日志
- `_ptr_robot->StopMotion()` 停止机器人运动
- `_ptr_robot->CloseRPC()` 断开 RPC 连接
- `_ptr_robot.release()` 释放指针
返回 SUCCESS。
##### 7️⃣ read() —— “从真机读状态 → 写进 state 内存（控制器读取）”
```
error_t returncode = _ptr_robot->GetActualJointPosDegree(1, &state_data);
```
- 成功：遍历 6 轴，把度转换为弧度写到 `_jnt_position_state[i]`  
    注释强调“moveit 统一用弧度”
- 失败：代码写了 `hardware_interface::return_type::ERROR;` 但**没有 return**，所以最后还是返回 OK。
##### 8️⃣ write() —— “从 command 内存取命令 → 通过 SDK 下发”
###### 8.1 位置控制模式 `_control_mode == 0`

**(1) 先检查命令是否是有限数 NaN/Inf**
```
std::any_of(&_jnt_position_command[0], &_jnt_position_command[5], [](double c){...})
```
防止控制器计算异常导致 NaN/Inf，下发会非常危险，所以直接 ERROR。

**(2) 组包 + 单位转换（弧度→度）**
`cmd.jPos[j] = _jnt_position_command[j]/M_PI*180;`

**(3) 下发 ServoJ**
- `ServoJ(&cmd,&extcmd,0,0,0.008,0,0)`
其中 `0.008`（8ms）非常关键：代表伺服指令时间间隔/控制节拍

**(4) 返回码检查**
returncode != 0 打印“指令下发错误”
###### 8.2 扭矩控制模式 `_control_mode == 1`（预留）
###### 8.3 未知模式

##### 9️⃣ 插件导出 —— controller_manager 为什么能加载到它？
```
#include "pluginlib/class_list_macros.hpp"
PLUGINLIB_EXPORT_CLASS(fairino_hardware::FairinoHardwareInterface, hardware_interface::SystemInterface)
```
- 把你的类注册为 pluginlib 插件入口
- controller_manager 才能通过 `fairino_hardware.xml` 中的 type 找到并实例化它
##### 🔟 把整份 .cpp 的运行过程串起来
启动时：
1. controller_manager 加载插件 → 调 `on_init()` 检查接口（只允许 position）
2. 调 `export_state_interfaces()` / `export_command_interfaces()`：把 `_jnt_position_state/_jnt_position_command` 指针交给系统
3. 切换到 active → 调 `on_activate()`：连 RPC、读真实关节角、command=state 对齐
控制循环每一拍：
4. `read()`：SDK 读真实关节角（度→弧度）写入 `_jnt_position_state[]`
5. 控制器（如 joint_trajectory_controller）根据轨迹插补，写入 `_jnt_position_command[]`
6. `write()`：从 `_jnt_position_command[]` 取值（弧度→度），用 ServoJ 发给控制器

停止时：
7. `on_deactivate()`：StopMotion → CloseRPC → release

### 第 2 类：插件描述文件（pluginlib 必需）
#### fairino_hardware.xml
👉 **“告诉 ROS：这个 .so 里有什么插件类”**
如果没有它：
- 编译没问题
- 运行时 controller_manager **找不到你的硬件接口**

```
<library path="fairino_hardware">
  <class
      name="fairino_hardware/FairinoHardwareInterface"
      type="fairino_hardware::FairinoHardwareInterface"
      base_class_type="hardware_interface::SystemInterface">
    <description>
      Hardware interface for Fairino Robots
    </description>
  </class>
</library>
```

##### 1️⃣ 这个文件是给谁看的？

👉 **给 pluginlib / controller_manager 看**
当 controller_manager 启动时，它会：
###### 1.从 URDF 的 `<ros2_control>` 中读到：
```
plugin: fairino_hardware/FairinoHardwareInterface
```
###### 2.pluginlib 去查 **所有已安装的 plugin XML**
###### 3.在 `fairino_hardware.xml` 里找到：
```
name="fairino_hardware/FairinoHardwareInterface"
```

##### 2️⃣ 每一项你必须理解清楚
==library path="fairino_hardware"==
表示最终生成的动态库名是：libfairino_hardware.so
必须和 CMakeLists.txt 里 add_library 的名字一致

==name="fairino_hardware/FairinoHardwareInterface"==
- 这是 **URDF 里填写的 plugin 名字**
- MoveIt / ros2_control **只认这个字符串**

==type="fairino_hardware::FairinoHardwareInterface"`====
- C++ 中的 **完整类名（命名空间 + 类名）**
- 必须和你在 `.hpp/.cpp` 里定义的一模一样
    
==base_class_type="hardware_interface::SystemInterface"==
- 告诉 pluginlib：  
    👉 **这个插件是一个 ros2_control 硬件接口**

controller_manager 就是通过这个基类，知道：
- 可以调用 `on_init`
- 可以调用 `read/write`
- 可以进入生命周期管理


### 第 3 类：构建与可见性支持（辅助但必要）
#### visibility_control.h
- 控制动态库符号导出
- 保证 pluginlib 在 Linux / Windows 下都能加载类