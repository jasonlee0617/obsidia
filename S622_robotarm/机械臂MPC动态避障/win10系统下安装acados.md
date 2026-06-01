本项目已验证通过的环境如下：

| 项目 | 版本 / 路径 |
|---|---|
| 操作系统 | Windows 10 |
| MATLAB | MATLAB R2023b |
| acados | acados v0.3.6 |
| Git | Git-2.54.0-64-bit |
| CMake | CMake 4.3.2 |
| 编译器 | MinGW64 GCC 8.1.0 |
| MATLAB MEX 编译器 | MinGW64 Compiler |
| 求解器后端 | acados |
| 项目根目录 | `C:/Users/JasonLee/OneDrive/Desktop/s622arm_matlab/S622ARM_Workspace` |

当前项目中的第三方依赖统一放在：
```
C:/Users/JasonLee/OneDrive/Desktop/s622arm_matlab/S622ARM_Workspace/third_party
```

目录结构为：
```
S622ARM_Workspace/
├─ demo_MPC_BIRRTstar_hybrid_optimized.m
├─ setup_acados_env.m
├─ install_acados_third_party.m
├─ fix_acados_install_escape.m
├─ kinematics/
├─ planning/
├─ mpc/
├─ mpc_solver/
│  ├─ solveMPC_acados.m
│  ├─ solveMPC_dispatch.m
│  ├─ NLP_utils/
│  │  └─ buildAcadosNLP.m
│  └─ c_generated_code/
├─ obstacle/
├─ utils/
└─ third_party/
   ├─ acados_v036/
   ├─ casadi/
   └─ matlab_support_packages/
      └─ mingw64/

```

# 1.总体部署逻辑

acados 部署到 MATLAB 的完整流程为：
```
准备 third_party 目录
        ↓
配置 MinGW
        ↓
配置 MATLAB MEX 使用 MinGW
        ↓
配置 CasADi
        ↓
编译安装 acados
        ↓
如遇 Windows 路径转义错误，运行 fix_acados_install_escape.m
        ↓
运行 acados 官方 minimal_example_ocp 验证
        ↓
修改并验证 buildAcadosNLP.m
        ↓
运行 demo_MPC_BIRRTstar_hybrid_optimized.m

```

# 2.准备 acados v0.3.6

下载 acados，应在 `third_party` 目录下执行：
```
cd /d C:\Users\JasonLee\OneDrive\Desktop\s622arm_matlab\S622ARM_Workspace\third_party

git clone --branch v0.3.6 --recursive https://github.com/acados/acados.git acados_v036

cd acados_v036
git submodule update --init --recursive

```

检查版本：
```
git describe --tags --always
```

期望输出：
```
v0.3.6

```

检查子模块：
```
git submodule status --recursive
```

# 3.准备 MinGW
MinGW 当前路径为：
```
S622ARM_Workspace/third_party/matlab_support_packages/mingw64
```

应包含：
```
mingw64/
├─ bin/
│  ├─ gcc.exe
│  ├─ g++.exe
│  └─ mingw32-make.exe

```

MATLAB 中检查：
```
clear; clc;

projectRoot = "C:/Users/JasonLee/OneDrive/Desktop/s622arm_matlab/S622ARM_Workspace";
mingw_root = fullfile(projectRoot, "third_party", "matlab_support_packages", "mingw64");
mingw_bin  = fullfile(mingw_root, "bin");

mingw_root = strrep(char(mingw_root), '\', '/');
mingw_bin  = strrep(char(mingw_bin),  '\', '/');

setenv('MW_MINGW64_LOC', mingw_root);
setenv('PATH', [mingw_bin pathsep getenv('PATH')]);

system('where gcc')
system('where g++')
system('where mingw32-make')
system('gcc --version')

```

期望输出中出现：
```
.../third_party/matlab_support_packages/mingw64/bin/gcc.exe
.../third_party/matlab_support_packages/mingw64/bin/g++.exe
.../third_party/matlab_support_packages/mingw64/bin/mingw32-make.exe

```

# 4.配置 MATLAB MEX 使用 MinGW

移动 MinGW 到 `third_party` 后，需要重新配置 MATLAB MEX 编译器。

MATLAB 中执行：
```
clear; clc;

projectRoot = "C:/Users/JasonLee/OneDrive/Desktop/s622arm_matlab/S622ARM_Workspace";
mingw_root = fullfile(projectRoot, "third_party", "matlab_support_packages", "mingw64");
mingw_bin  = fullfile(mingw_root, "bin");

mingw_root = strrep(char(mingw_root), '\', '/');
mingw_bin  = strrep(char(mingw_bin),  '\', '/');

setenv('MW_MINGW64_LOC', mingw_root);
setenv('PATH', [mingw_bin pathsep getenv('PATH')]);

mex -setup C
mex -setup C++

```

选择：
```
MinGW64 Compiler (C)
MinGW64 Compiler (C++)
```

检查：
```
mex.getCompilerConfigurations('C','Selected')
mex.getCompilerConfigurations('C++','Selected')
```

期望：
```
Name: 'MinGW64 Compiler (C)'
Name: 'MinGW64 Compiler (C++)'
Location: '...\third_party\matlab_support_packages\mingw64'
```

# 5.测试 MEX 编译器

```
clear; clc;

testDir = fullfile(tempdir, 'mex_mingw_test');
if ~isfolder(testDir)
    mkdir(testDir);
end

cd(testDir);

copyfile(fullfile(matlabroot, 'extern', 'examples', 'mex', 'yprime.c'), '.', 'f');

mex yprime.c

yprime(1, 1:4)

```

成功标志：
```
使用 'MinGW64 Compiler (C)' 编译。
MEX 已成功完成。

```

示例输出：
```
2.0000    8.9685    4.0000   -1.0947
```

# 6.配置 CasADi

CasADi 应放在：
```
S622ARM_Workspace/third_party/casadi
```

# 7.启动顺序

重新编译acados依赖
```
运行install_acados_third_party.m
```

出现红色警告报错：
```
运行fix_acados_install_escape.m
```

最后运行：
```
demo_MPC_BIRRTstar_hybrid_optimized.m
```