# 1.依赖安装

```
sudo apt update
sudo apt install -y \
    build-essential cmake git \
    gcc g++ gfortran \
    liblapack-dev libblas-dev libopenblas-dev \
    python3 python3-pip

```

验证版本：
```
gcc --version      # >= 9
cmake --version    # >= 3.16
git --version

```

# 2.下载 acados（含子模块）

## 2.1 克隆仓库

```
# 选择安装目录（示例：项目工作空间内）
cd /home/robot/桌面/acados_toolkit

git clone https://github.com/acados/acados.git
cd acados

```

## 2.2 拉取子模块（关键步骤）

```
git checkout e0759960cae76e4f2e8177cf2866a6ab088f4e86
git submodule update --init --recursive
```

## 2.3 验证子模块完整性
```
ls external/blasfeo/CMakeLists.txt   # 必须存在
ls external/hpipm/CMakeLists.txt     # 必须存在

```

# 3、编译 acados

## 3.1 创建构建目录并执行 CMake

```
cd /home/robot/桌面/acados_toolkit/acados

mkdir -p build && cd build

cmake .. \
    -DCMAKE_INSTALL_PREFIX="/home/robot/桌面/acados_toolkit/acados" \
    -DCMAKE_BUILD_TYPE=Release \
    -DACADOS_WITH_QPOASES=ON \
    -DACADOS_WITH_OSQP=OFF \
    -DACADOS_WITH_DAQP=OFF \
    -DBLASFEO_TARGET=X64_AUTOMATIC \
    -DHPIPM_TARGET=X64_AUTOMATIC \
    -DACADOS_INSTALL_DIR="/home/robot/桌面/acados_toolkit/acados"

```

## 3.2 编译与安装

```
make -j$(nproc)
make install
```

## 3.3 验证编译产物

```
cd /home/robot/桌面/acados_toolkit/acados
# 动态库
ls -la lib/
# 预期:
#   libacados.so
#   libblasfeo.so
#   libhpipm.so
#   libqpOASES_e.so.3.1

# 头文件
ls include/acados/
# 预期:
#   dense_qp/  ocp_nlp/  ocp_qp/  sim/  utils/

# MATLAB 接口
ls interfaces/acados_matlab_octave/
# 预期:
#   AcadosOcp.m  AcadosOcpSolver.m  acados_template_mex/  ...

```

# 4、注册系统动态库

```
[!danger] 关键步骤  
MATLAB 内部的 `setenv('LD_LIBRARY_PATH', ...)` **对 MEX 动态链接器无效**。MEX 加载 `.so` 时用的是 MATLAB 启动前的系统 `LD_LIBRARY_PATH`。必须通过 `ldconfig` 注册。
```

## 4.1 创建库配置文件

```
sudo bash -c 'echo "/home/robot/桌面/acados_toolkit/acados/lib" > /etc/ld.so.conf.d/acados.conf'

```

## 4.2 刷新动态库缓存

```
sudo ldconfig

```

## 4.3 验证

```
ldconfig -p | grep acados

```

预期输出：
```
libacados.so (libc6,x86-64) => /home/robot/.../acados/lib/libacados.so

```

# 5、CasADi 安装

## 5.1 下载:
从 [CasADi Releases](https://github.com/casadi/casadi/releases) 下载 Linux MATLAB 版本：
```
cd /home/robot/桌面/acados_toolkit

# 下载 CasADi 3.6.7（已验证兼容）
wget https://github.com/casadi/casadi/releases/download/3.6.7/casadi-3.6.7-linux64-matlab2018b.zip

# 解压
mkdir -p casadi
unzip casadi-3.6.7-linux64-matlab2018b.zip -d casadi/

```

## 5.2 验证

在 MATLAB 中：
```
addpath('/home/robot/桌面/s622arm_matlab/S622ARM_WORKSPACE/casadi');
import casadi.*
x = SX.sym('x');
disp(CasadiMeta.version());  % 应输出 3.6.7

```

# 6、MATLAB MEX 编译器配置

在 MATLAB 命令窗口中执行：
```
mex -setup C
mex -setup C++

```

预期输出：
```
MEX 配置为使用 'gcc' 以进行 C 语言编译。
MEX 配置为使用 'g++' 以进行 C++ 语言编译。

```

# 7、MATLAB 环境配置脚本

创建 `setup_acados_env.m`，放在项目根目录：
```
function setup_acados_env()
%SETUP_ACADOS_ENV 配置 acados + CasADi 环境
acados_dir = '/home/robot/桌面/acados_toolkit/acados';
casadi_dir = '/home/robot/桌面/acados_toolkit/casadi';
% 1. CasADi（必须在 acados 之前加载）
if exist(casadi_dir, 'dir')
addpath(casadi_dir);
end

% 2. 环境变量
setenv('ACADOS_INSTALL_DIR', acados_dir);
setenv('ENV_RUN', 'true');

% 3. 动态库路径
lib_path = fullfile(acados_dir, 'lib');
cgen_path = fullfile(acados_dir, '..', 'c_generated_code');
current_ld = getenv('LD_LIBRARY_PATH');
if ~contains(current_ld, lib_path)
setenv('LD_LIBRARY_PATH', [lib_path ':' current_ld]);
end

current_ld = getenv('LD_LIBRARY_PATH');
if ~contains(current_ld, cgen_path)
setenv('LD_LIBRARY_PATH', [cgen_path ':' current_ld]);
end

% 4. MATLAB 接口路径
matlab_if = fullfile(acados_dir, 'interfaces', 'acados_matlab_octave');
addpath(matlab_if);
subdirs = {'acados_template_mex'};
for i = 1:numel(subdirs)
d = fullfile(matlab_if, subdirs{i});
if exist(d, 'dir'), addpath(d); end
end

% 5. 验证
if exist('AcadosOcp', 'class') || exist('AcadosOcp.m', 'file')
fprintf('[OK] 新版 API (AcadosOcp/AcadosOcpSolver) 可用\n');
end

if exist(fullfile(lib_path, 'libacados.so'), 'file')
fprintf('[OK] libacados.so 找到\n');
end

fprintf('[OK] acados 环境配置成功: %s\n', acados_dir);
end

```

# 8、验证部署

## ## 8.1 创建验证脚本

创建 `test_acados_simple.m`：

```
%% acados 部署验证脚本
%% 简单测试：确认 AcadosOcpSolver 可以创建和求解

clear all; close all; clc;
% ===== [关键] 先加载 CasADi =====
addpath('/home/robot/桌面/acados_toolkit/casadi');
% 验证 CasADi 可用
try
import casadi.*
x_test = SX.sym('x_test');
fprintf('[OK] CasADi 可用, 版本: %s\n', CasadiMeta.version());
catch ME
error('[FAIL] CasADi 不可用: %s', ME.message);
end
% 加载 acados 环境
setup_acados_env();
%% 1. 一维 double integrator
x = SX.sym('x', 2);
u = SX.sym('u');
ocp = AcadosOcp();
ocp.model.name = 'test_1d';
ocp.model.x = x;
ocp.model.u = u;
ocp.model.f_expl_expr = [x(2); u];
ocp.cost.cost_type = 'EXTERNAL';
ocp.cost.cost_type_e = 'EXTERNAL';
ocp.model.cost_expr_ext_cost = 10*(x(1)-1)^2 + (x(2))^2 + 0.1*u^2;
ocp.model.cost_expr_ext_cost_e = 100*(x(1)-1)^2 + 10*(x(2))^2;
N = 20; dt = 0.05;
ocp.constraints.x0 = [0; 0];
ocp.constraints.idxbu = int32(0);
ocp.constraints.lbu = -5;
ocp.constraints.ubu = 5;
ocp.solver_options.tf = N * dt;
ocp.solver_options.N_horizon = N;
ocp.solver_options.nlp_solver_type = 'SQP';
ocp.solver_options.qp_solver = 'PARTIAL_CONDENSING_HPIPM';
ocp.solver_options.integrator_type = 'ERK';
ocp.solver_options.print_level = 0;

%% 2. 创建求解器

fprintf('正在创建 AcadosOcpSolver...\n');
solver = AcadosOcpSolver(ocp);
fprintf('求解器创建成功！\n');
%% 3. 求解
solver.solve();
stat = solver.get('status');
fprintf('求解状态: %d (0=成功)\n', stat);

%% 4. 提取结果
x_traj = zeros(N+1, 2);
u_traj = zeros(N, 1);
for k = 0:N
x_traj(k+1, :) = solver.get('x', k)';
end

for k = 0:N-1
u_traj(k+1) = solver.get('u', k);
end

%% 5. 画图
t_x = (0:N) * dt;
t_u = (0:N-1) * dt;
figure('Name', 'acados 测试');
subplot(3,1,1); plot(t_x, x_traj(:,1), 'b-o'); ylabel('位置');
yline(1, 'r--', '目标'); title('acados 测试通过！');
subplot(3,1,2); plot(t_x, x_traj(:,2), 'g-o'); ylabel('速度');
subplot(3,1,3); stairs(t_u, u_traj, 'k-'); ylabel('控制'); xlabel('时间 (s)');
fprintf('\n===== acados 部署验证通过！=====\n');
fprintf('位置终值: %.4f (目标 1.0)\n', x_traj(end,1));
fprintf('速度终值: %.4f (目标 0.0)\n', x_traj(end,2));

```

## 8.2 运行验证
```
cd('/home/robot/桌面/acados_toolkit');
test_acados_simple
```

## 8.3 预期输出

```
[OK] CasADi 可用, 版本: 3.6.7
[OK] 新版 API (AcadosOcp/AcadosOcpSolver) 可用
[OK] libacados.so 找到
[OK] acados 环境配置成功: .../acados
正在创建 AcadosOcpSolver...
[OK] 求解器创建成功！
[OK] 求解状态: 0 (0=成功)

===== acados 部署验证通过！=====
位置终值: 0.9844 (目标 1.0)
速度终值: 0.0533 (目标 0.0)

```


输出如下：
```
出错 test_1d_mex_solver (第 50 行) obj.C_ocp = acados_mex_create_test_1d(); 出错 AcadosOcpSolver (第 171 行) obj.t_ocp = mex_solver(obj.solver_creation_opts); 出错 test_acados_simple (第 49 行) solver = AcadosOcpSolver(ocp);
```

解决办法：
cd /home/robot/桌面/acados_toolkit路径下创建run_matlab.sh
```
#!/bin/bash
# 启动 MATLAB 并设置 acados 所需的动态库路径
export ACADOS_INSTALL_DIR=/home/robot/桌面/acados_toolkit/acados
export LD_LIBRARY_PATH=/home/robot/桌面/s622arm_matlab/S622ARM_WORKSPACE/c_generated_code:/home/robot/桌面/acados_toolkit/acados/lib:$LD_LIBRARY_PATH
export ENV_RUN=true
/home/robot/Matlabwork/bin/matlab "$@"
```
退出现在的matlab,以下方法启动matab

执行：
```

~/桌面/s622arm_matlab/run_matlab.sh &
```
具体路径要对应修改
或者：
```
bash /home/robot/桌面/s622arm_matlab/S622ARM_WORKSPACE/run_matlab.sh
```
继续测试`test_acados_simple.m`

## 8.4项目集成要点

```
项目根目录/
├── mpc_solver/
│   ├── buildAcadosNLP.m        # 构建 OCP（返回 AcadosOcpSolver）
│   └── solveMPC_acados.m       # MPC 求解（调用 solver.set/solve/get）
├── setup_acados_env.m          # 环境配置
├── test_acados_simple.m        # 验证脚本
└── c_generated_code/           # 自动生成（首次运行后出现）

```

```
├── acados/                     # acados 安装目录
│   ├── lib/                    # 动态库
│   ├── include/                # 头文件
│   └── interfaces/             # MATLAB 接口
│       └── acados_matlab_octave/
├── casadi/                     # CasADi 库
```
## 8.4 清理与重新生成

```
# 清理自动生成的代码（遇到奇怪错误时执行）
cd /home/robot/桌面/acados_toolkit/acados/build
git reset --hard
git clean -fdx
git fetch origin
cd /home/robot/桌面/acados_toolkit/acados
rm -rf build
```



# 9 fairino_mpc_avoidance的acados依赖构建

依赖准备：

安装python版本的acados_template，方便迁移到moveit框架
```
cd /home/robot/桌面/acados_toolkit/acados
python3 -m pip install casadi==3.7.2
python3 -m pip install deprecated
python3 -m pip install -e ./interfaces/acados_template
```

再确认 Python 导入路径：
```
python3 -c "import acados_template; print(acados_template.__file__)"
```

应输出类似：
```
/home/robot/桌面/acados_toolkit/acados/interfaces/acados_template/acados_template/__init__.py
```

确认节点链接动态库：
```
ldd /home/robot/S622_robotarm/install/fairino_mpc_avoidance/lib/fairino_mpc_avoidance/mpc_avoidance_node | grep acados
```

应看到：
```
/home/robot/桌面/acados_toolkit/acados/lib/libacados.so
```

如果修改了 OCP 模型，需要重新生成 acados solver：
```bash
source /opt/ros/humble/setup.bash
source /home/robot/S622_robotarm/src/fairino_mpc_avoidance/setup_acados_env.sh
python3 ~/S622_robotarm/src/fairino_mpc_avoidance/scripts/generate_acados_solver.py 
colcon build --packages-select fairino_mpc_avoidance --symlink-install
source /home/robot/S622_robotarm/install/setup.bash
ros2 launch fairino_mpc_avoidance mpc_planning_demo.launch.py
```


## acados 问题的最终解决

acados依赖路径永久写进系统环境中

```
echo 'export ACADOS_INSTALL_DIR=/home/robot/Desktop/acados_toolkit/acados' >> ~/.bashrc echo 'export LD_LIBRARY_PATH=/home/robot/Desktop/acados_toolkit/acados/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
```

最终方案是让三者完全统一到同一棵目录：
```
/home/robot/桌面/acados_toolkit/acados
```

统一后的关系是：
```
ACADOS_INSTALL_DIR
→ /home/robot/桌面/acados_toolkit/acados

acados_template
→ /home/robot/桌面/acados_toolkit/acados/interfaces/acados_template

libacados.so
→ /home/robot/桌面/acados_toolkit/acados/lib/libacados.so

libhpipm.so
→ /home/robot/桌面/acados_toolkit/acados/lib/libhpipm.so

libblasfeo.so
→ /home/robot/桌面/acados_toolkit/acados/lib/libblasfeo.so
```