### 1.QOS通信策略
#### 1.1rgb、depth订阅
```
self.rgb_sub = Subscriber(self, Image, self.rgb_topic, qos_profile=qos_profile_sensor_data)
        self.depth_sub = Subscriber(self, Image, self.depth_topic, qos_profile=qos_profile_sensor_data)
        self.sync = ApproximateTimeSynchronizer([self.rgb_sub, self.depth_sub],queue_size=self.sync_queue_size,slop=self.sync_slop,allow_headerless=False)
self.sync.registerCallback(self.synced_rgb_depth_callback)
只处理同步后的 RGB+Depth：
        - 只做：转换 -> 同一把锁内同时写 latest_rgb/latest_depth
```

#### 1.2控制发布者策略
```
qos_reliable_latest = QoSProfile(history=HistoryPolicy.KEEP_LAST,depth=1,reliability=ReliabilityPolicy.RELIABLE,durability=DurabilityPolicy.VOLATILE)#控制发布者/订阅者的通信策略
        #history=HistoryPolicy.KEEP_LAST(历史策略：告诉 DDS（ROS2 底层通信）如何保存消息历史)
        #队列深度（缓存条数,depth=1 表示：只保留最新消息
        #durability=DurabilityPolicy.VOLATILE,订阅者刚启动时，只会收到启动之后发布的消息
        # Publishers
        self.pub_vis = self.create_publisher(Image, '/camera/detected_image', qos_reliable_latest)

        self.pub_pen_position = self.create_publisher(PointStamped, '/pen_position_3d', qos_reliable_latest)
        self.pub_box_position = self.create_publisher(PointStamped, '/box_position_3d', qos_reliable_latest)
        self.pub_cube_position = self.create_publisher(PointStamped, '/cube_position_3d', qos_reliable_latest)

        self.pub_pen_rpy = self.create_publisher(Float32MultiArray, '/pen_rpy', qos_reliable_latest) if self.publish_rpy else None
        self.pub_box_rpy = self.create_publisher(Float32MultiArray, '/box_rpy', qos_reliable_latest) if self.publish_rpy else None
        self.pub_cube_rpy = self.create_publisher(Float32MultiArray, '/cube_rpy', qos_reliable_latest) if self.publish_rpy else None
```

### 2.pen、cube（rpy欧拉角的获取）
#### 2.1限制pen rpy角度[0-pi],cube rpy角度[0 pi/2]方便计算抓取姿态
```
def yaw_0_to_pi_right0_left180(corners_2d: np.ndarray) -> float:
    """
    从OBB角点计算偏航角（范围：0到π）
    
    物理意义：
    0°: 物体右侧朝右（最长的边水平向右）
    90°: 物体右侧朝下
    180°: 物体右侧朝左
    
    参数：
    corners_2d: 4×2数组，OBB的四个角点坐标
    
    返回：
    偏航角（弧度），范围[0, π]
    """
    c = corners_2d.astype(np.float32)
    best_v = None
    best_len = -1.0
    # 遍历四条边，找出最长的边
    for i in range(4):
        v = c[(i + 1) % 4] - c[i]  # 计算第i条边向量：从角点i到角点(i+1)%4
        L = float(np.linalg.norm(v)) # 计算边的长度（L2范数）
        # 更新最长边
        if L > best_len:
            best_len = L
            best_v = v

    if best_v is None or best_len < 1e-6:
        return 0.0

    dx, dy = float(best_v[0]), float(best_v[1])    # 提取边向量的x,y分量
    yaw = math.atan2(abs(dy), dx)  # [0, pi]， 计算角度：使用atan2(|dy|, dx)将角度限制在[0, π]
    yaw = max(0.0, min(math.pi, yaw)) # 确保角度在[0, π]范围内
    return float(yaw)
```

#### 2.2解决ryz角度跳变
```
def wrap_to_pi(a: float) -> float:  # 包装后的角度，范围[-π, π]
    return float((a + math.pi) % (2.0 * math.pi) - math.pi)


def angle_diff(a: float, b: float) -> float:  #计算两个角度之间的最小差值
    return wrap_to_pi(a - b)


def choose_equivalent_angle(cur: float, prev: float, period: float) -> float:
    """
    参数：
    cur: 当前测量角度（已包装到[-π, π]）
    prev: 前一个角度
    period: 周期（pen: π, box/cube: π/2）
    
    返回：
    最接近prev的等价角度
    """

    best = cur
    best_err = abs(angle_diff(cur, prev))
    # 尝试多个周期偏移（-4到4个周期）
    for k in (-4, -3, -2, -1, 0, 1, 2, 3, 4):
        cand = cur + k * period # 生成候选角度
        err = abs(angle_diff(cand, prev)) # 计算与prev的差值
        if err < best_err:
            best_err = err
            best = cand
    return wrap_to_pi(best)
```

**pen、cube的不同处理策略**
```
    def _estimate_yaw_0_pi(self, cls: int, corners: np.ndarray) -> float:
        """
        估计偏航角（0到π范围）并进行平滑处理
        """
        # 1. 从角点计算偏航角
        yaw_meas = yaw_0_to_pi_right0_left180(corners)  # [0, pi]
        # 2. 获取前一个角度
        prev = self.prev_yaw.get(cls, None)
        # 3. 确定角度周期
        # pen: 周期π（180°），因为笔有方向性
        # box/cube: 周期π/2（90°），因为正方形旋转90°后看起来相同
        if cls in (1, 2):   # box, cube
            period = (math.pi / 2.0)
        else:               # pen
            period = math.pi
        # 4. 处理第一个测量值
        if prev is None:
            yaw_out = yaw_meas
        else:

            # 5. 将当前测量值转换到等价表示范围
            # 由于yaw_meas在[0,π]，而choose_equivalent_angle期望[-π,π]
            meas_rep = yaw_meas
            if meas_rep > (math.pi / 2.0):
                meas_rep = meas_rep - math.pi  # (-pi/2, pi/2]

            prev_rep = prev
            if prev_rep > (math.pi / 2.0):
                prev_rep = prev_rep - math.pi
            # 6. 选择最接近前一个角度的等价角度
            yaw_eq = choose_equivalent_angle(meas_rep, prev_rep, period=period)
            # 7. 计算角度差并应用低通滤波
            diff = angle_diff(yaw_eq, prev_rep)
            yaw_smooth_rep = wrap_to_pi(prev_rep + self.alpha * diff)

            # 8. 转换回[0,π]范围
            yaw_out = yaw_smooth_rep
            if yaw_out < 0.0:
                yaw_out += math.pi
        # 9. 确保角度在[0,π]范围内
        yaw_out = max(0.0, min(math.pi, float(yaw_out)))
        # 10. 存储当前角度供下次使用
        self.prev_yaw[cls] = yaw_out
        return yaw_out
```

### 3.计算pen、cube、box的姿态策略（掩膜采样策略）
```
    #   从OBB多边形和深度图像计算3D中心点
    def _center3d_from_obb_depth(self, poly_2d: np.ndarray, depth: np.ndarray):
        # 获取图像尺寸
        H, W = depth.shape[:2]
        # ========================
        # 1. 创建OBB掩膜
        # ========================
        mask = np.zeros((H, W), dtype=np.uint8)
        cv2.fillPoly(mask, [poly_2d.astype(np.int32)], 255) # 将多边形填充为白色（255）
        # ========================
        # 2. 获取多边形内的像素坐标
        # ========================
        ys, xs = np.where(mask > 0)# 掩膜内所有像素的坐标
        if xs.size < 100:# 检查是否有足够多的像素
            return None
        # ========================
        # 3. 提取相机内参
        # ========================
        fx, fy = self.camera_intrinsics['fx'], self.camera_intrinsics['fy']
        cx, cy = self.camera_intrinsics['cx'], self.camera_intrinsics['cy']
        # ========================
        # 4. 采样深度点并转换为3D点
        # ========================
        stride = max(1, self.stride) # 采样步长，避免处理过多点
        pts = []# 存储3D点
        count = 0
        for u, v in zip(xs[::stride], ys[::stride]):  # 遍历掩膜内的像素（按步长采样）
            Z = float(depth[v, u]) # 获取深度值
            if np.isfinite(Z) and 0.0 < Z <= self.depth_max_range: # 检查深度值的有效性
                # 从像素坐标和深度计算3D坐标
                X = (float(u) - cx) * Z / fx
                Y = (float(v) - cy) * Z / fy
                pts.append([X, Y, Z])
                count += 1
                # 达到最大点数限制时停止
                if count >= self.max_points:
                    break
        # ========================
        # 5. 检查是否有足够多的有效点
        # ========================
        if len(pts) < self.min_points:
            return None
        # ========================
        # 6. 计算所有3D点的质心
        # ========================
        P = np.asarray(pts, dtype=np.float32) # N×3数组
        return np.mean(P, axis=0) # 计算均值
```