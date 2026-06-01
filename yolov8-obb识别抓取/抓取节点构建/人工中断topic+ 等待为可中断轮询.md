### 1.抓取节点补丁
#### 1.1头部 import：新增线程/Bool/CallbackGroup
```
# --- PATCH: add imports ---
import threading
from std_msgs.msg import Bool
from rclpy.callback_groups import MutuallyExclusiveCallbackGroup
```

#### 1.2`__init__`：新增 callback groups、abort 订阅、加快 control_loop timer

在 `__init__` 里（`self.callback_group = ReentrantCallbackGroup()` 后面加入
```
 # control_loop 不允许重入（状态机安全）
 self.control_cb_group = MutuallyExclusiveCallbackGroup()
 # abort 回调必须能在运动阻塞时也执行（与 control_loop 分组隔离）
 self.abort_cb_group = MutuallyExclusiveCallbackGroup()
 
  # --- PATCH: abort event ---
 self.abort_event = threading.Event()
 self.abort_reason = ""
 
 # --- PATCH: subscribe manual abort ---
self.create_subscription(Bool,"/manual_abort",self._on_manual_abort,10,callback_group=self.abort_cb_group,)
# --- PATCH: faster control loop ---
self.create_timer(0.2, self.control_loop, callback_group=self.control_cb_group)

```

#### 1.3新增：取消执行 + 可中断等待 + recover 函数

在`PenCubeBoxGraspingNode` 类中（# ---------------- Motion ----------------后）创建：
```
 # =========================
    # --- PATCH: MANUAL ABORT ---
    # =========================
    def _on_manual_abort(self, msg: Bool):
        """
        /manual_abort 订阅回调函数：
        - 当收到 Bool(data=True) 时触发
        - 置位 abort_event（用于让正在等待/执行的逻辑尽快退出）
        - 立刻发送 cancel/stop，尽可能让机械臂立刻停下来
        """
        if not msg.data:
            return
        self.abort_reason = "manual abort (space)"
        # 置位中断事件：所有等待函数、状态机都应该优先检测它
        self.abort_event.set()
        self.get_logger().warn("!!! Manual abort requested !!!")
        # 关键：立刻发 stop/cancel，不要等 control_loop
        self._cancel_all_motion_now()

    def _cancel_moveit(self, m, name: str):
        """
        双保险取消：
        1) pymoveit2 的 cancel_execution()（发布 /trajectory_execution_event: 'stop'）
        2) 直接 cancel 当前 action goal（访问内部 goal_handle）
        """
        # 1) 走 pymoveit2 的 stop 机制
        try:
            m.cancel_execution()
        except Exception as e:
            self.get_logger().warn(f"[{name}] cancel_execution failed: {e}")

        # 2) 直接 cancel action goal_handle
        try:
            mutex = getattr(m, "_MoveIt2__execution_mutex", None)
            if mutex:
                mutex.acquire()
            gh = getattr(m, "_MoveIt2__execution_goal_handle", None)
            if mutex:
                mutex.release()

            if gh is not None:
                gh.cancel_goal_async()
        except Exception as e:
            self.get_logger().warn(f"[{name}] goal_handle cancel failed: {e}")

    def _cancel_all_motion_now(self):
        """
        取消所有运动（arm + gripper）：
        """
        self.get_logger().warn("Stopping arm & gripper...")
        try:
            self._cancel_moveit(self.moveit2_arm, "arm")
        except Exception:
            pass
        try:
            self._cancel_moveit(self.moveit2_gripper, "gripper")
        except Exception:
            pass

        # 如果未来在 ros2_control 侧暴露 StopMotion 服务，可在这里加服务调用
        # self._call_stop_motion_service()

    def _wait_moveit_idle_or_abort(self, m, action_name: str, timeout_sec: float) -> bool:
        """
        代替 wait_until_executed()：
        - 轮询 m.query_state()
        - abort_event 触发时立即 cancel 并返回 False
        """
        t0 = time.time()
        while rclpy.ok():
            # ---------- 1) abort 优先 ----------
            if self.abort_event.is_set():
                self.get_logger().warn(f"ABORT while waiting: {action_name}")
                self._cancel_all_motion_now()
                return False
            # ---------- 2) 查询执行状态 ----------
            try:
                st = m.query_state()   # 返回 MoveIt2State enum
                if hasattr(st, "value"):
                    st_val = st.value
                else:
                    st_val = int(st)
            except Exception:
                st_val = 0

            # MoveIt2State.IDLE == 0,说明已经结束（成功/失败看 motion_suceeded）
            if st_val == 0:
                # pymoveit2 用 motion_suceeded 记录结果
                return bool(getattr(m, "motion_suceeded", False))

            if (time.time() - t0) > timeout_sec:
                self.get_logger().error(f"{action_name} timeout -> force stop")
                self._cancel_all_motion_now()
                # 最后兜底：避免内部状态卡死导致后续无法执行
                try:
                    m.force_reset_executing_state()
                except Exception:
                    pass
                return False

            time.sleep(0.02)  # 50Hz 轮询

        return False

    def _recover_from_abort(self):
        self.get_logger().warn(f"=== RECOVER FROM ABORT: {self.abort_reason} ===")

        # 确保停住
        self._cancel_all_motion_now()

        # keepout 状态清理
        try:
            if self.keepout_enabled:
                self.disable_keepout()
        except Exception:
            pass

        # 打开夹爪更安全
        try:
            self.control_gripper(False)
        except Exception:
            pass
        
        try:
            self.moveit2_arm.max_velocity = 0.15
            self.moveit2_arm.max_acceleration = 0.15
        except Exception:
            pass

        # 回 home
        ok_home = False
        try:
            ok_home = self.go_home()
        except Exception:
            ok_home = False

        # 清空缓存，重新开始检测
        self.active_target = None
        self.poses = {}
        self.target_pen_position = None
        self.target_cube_position = None
        self.target_box_position = None
        self.target_pen_rpy = None
        self.target_cube_rpy = None

        # 清理 abort 标志
        self.abort_event.clear()
        self.abort_reason = ""

        # 直接回到 SEARCHING
        if ok_home:
            self.current_state = TaskState.SEARCHING
            return
```

#### 1.4修改三处函数`move_to_pose()` / `control_gripper()` / `go_home()`

```
ok = self.moveit2_arm.wait_until_executed()
ok = self.moveit2_gripper.wait_until_executed()
ok = self.moveit2_arm.wait_until_executed()
```

替换为：
```
ok = self._wait_moveit_idle_or_abort(self.moveit2_arm, action_name, timeout_sec=30.0)
ok = self._wait_moveit_idle_or_abort(self.moveit2_gripper, action, timeout_sec=10.0)
ok = self._wait_moveit_idle_or_abort(self.moveit2_arm, "Go HOME", timeout_sec=30.0)
```

#### 1.5`control_loop()` 开头抢占：一旦 abort_event 被置位，立即 recover

在 `def control_loop(self):` 开头加
```
# --- PATCH: abort has priority ---
if self.abort_event.is_set():
    self._recover_from_abort()
    return
```

### 2.键盘节点（按空格发布 /manual_abort）
#### 2.1创建stopmotion_node.py节点
```
#!/usr/bin/env python3
import sys, termios, tty, select
import rclpy
from rclpy.node import Node
from std_msgs.msg import Bool

class KeyboardAbort(Node):
    def __init__(self):
        super().__init__("keyboard_abort_space")
        # 创建发布者：发布 /manual_abort (Bool)
        self.pub = self.create_publisher(Bool, "/manual_abort", 10)

        # ========= 终端键盘读取准备 =========
        # sys.stdin.fileno()：获取标准输入的文件描述符
        # termios.tcgetattr：读取当前终端属性（用于后续恢复）
        # tty.setcbreak：设置成 cbreak 模式（按键无需回车即可读到）
        #stdin 必须是 TTY，否则 tcgetattr 会失败
        self.fd = sys.stdin.fileno()
        self.old = termios.tcgetattr(self.fd)
        tty.setcbreak(self.fd)    # 设置为“按键立即可读”的模式

        # 定时器：每 0.05s 检查一次键盘输入（20Hz）
        self.timer = self.create_timer(0.05, self.tick)
        self.get_logger().info("Press <SPACE> to ABORT (stop & go home).")

    def tick(self):
        """
        非阻塞检查是否有按键输入：
        - select.select([stdin], [], [], 0.0)：timeout=0 表示立刻返回
        - 如果 stdin 可读，读一个字符
        - 如果是空格 ' '，就发布 /manual_abort = True
        """
        r, _, _ = select.select([sys.stdin], [], [], 0.0)
        if not r:  # 没按键，直接返回，不阻塞
            return
        ch = sys.stdin.read(1)
        # 空格触发
        if ch == " ":
            self.pub.publish(Bool(data=True))
            self.get_logger().warn("ABORT sent: /manual_abort = true")

    def destroy_node(self):
        termios.tcsetattr(self.fd, termios.TCSADRAIN, self.old)
        super().destroy_node()

def main():
    rclpy.init()
    n = KeyboardAbort()
    try:
        rclpy.spin(n)
    finally:
        n.destroy_node()
        rclpy.shutdown()

if __name__ == "__main__":
    main()

```

#### 2.2创建stopmotion.launch.py节点
```
#!/usr/bin/env python3
from launch import LaunchDescription
from launch.actions import ExecuteProcess

def generate_launch_description():
    return LaunchDescription([
        ExecuteProcess(
            cmd=[
                "gnome-terminal", "--",
                "bash", "-lc",
                "ros2 run yolov8_grasping stopmotion"
            ],
            output="screen",
        ),
    ])

```

### 3.运行方式

先启动stopmotion.launch.py
```
ros2 launch yolov8_grasping stopmotion.launch.py 
```

![[Pasted image 20260111163546.png]]

后启动 ros2 launch yolov8_grasping  pen_box_system.launch.py 
