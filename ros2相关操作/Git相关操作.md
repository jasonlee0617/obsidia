# 1.初次Git上传步骤（命令行）

## 1.1进入工作区src

```
cd ~/工作区根目录/src
```

## 1.2配置 Git 用户名和邮箱

```
##git config --global user.name "你的GitHub用户名"
##git config --global user.email "你的GitHub邮箱"
git config --global user.name "jasonlee0617"  
git config --global user.email "676983492@qq.com"
```

检查是否配置成功：
```
git config --global user.name
git config --global user.email
```

正常输出应该是：
```
jasonlee0617
676983492@qq.com
```

## 1.3初始化 Git 仓库

在 `src` 目录下执行：
```
git init
```

然后把默认分支改成 `main`：
```
git branch -M main
```

## 1.4创建 `.gitignore` 文件

在 `src` 目录下执行：
```
touch .gitignore
code .gitignore
```

然后把下面内容复制进去：
```
# ROS2 build artifacts
build/
install/
log/

# Python cache
__pycache__/
*.pyc
*.pyo
*.pyd

# C/C++ build files
*.o
*.so
*.a
*.out

# CMake
CMakeFiles/
CMakeCache.txt
cmake_install.cmake
Makefile

# Colcon
.colcon/
.metadata/

# VS Code personal settings
.vscode/ipch/
.vscode/.browse.c_cpp.db*
.vscode/c_cpp_properties.json

# System files
.DS_Store
Thumbs.db

# Logs
*.log
```

## 1.5查看当前 Git 状态

```
git status
```
应该会看到很多红色文件，表示它们还没有被 Git 跟踪

## 1.6添加所有文件到暂存区

```
git add .
```

再次查看状态：
```
git status
```
这时文件应该变成绿色，表示已经准备提交

## 1.7提交代码

```
git commit -m "Initial commit: add ROS2 Humble S622 robot arm source packages"
```

如果成功，会看到类似：
```
[main xxxxxxx] Initial commit: add ROS2 Humble S622 robot arm source packages
```

## 1.8添加 GitHub 远程仓库

```
##git remote add origin git@github.com:你的用户名/你的仓库名.git
git remote add origin https://github.com/jasonlee0617/S622_robotarm_src.git
```

然后检查：
```
git remote -v
```

应该看到：
```
origin  https://github.com/jasonlee0617/S622_robotarm_src.git (fetch)
origin  https://github.com/jasonlee0617/S622_robotarm_src.git (push)
```

## 1.9上传到 GitHub

```
git push -u origin main
```

上传失败，重新初始化git:
```
##删除本地 Git 记录
rm -rf .git
```

然后重新初始化：
```
git init
git branch -M main

```

重新设置远程仓库：
```
##git remote add origin git@github.com:你的用户名/你的仓库名.git
git remote add origin https://github.com/jasonlee0617/S622_robotarm_src.git
```

```
git status
git add .
git commit -m "Initial commit: add ROS2 Humble S622 robot arm source packages"
git push -u origin main
```


## 1.10冲突文件解决

![[Pasted image 20260601145816.png]]


```
git pull --rebase origin main
```

# 2.日常同步本地代码到 GitHub 的标准流程

```
先确认状态 → 拉取远程更新 → 添加修改 → 提交 → 推送
```

```
cd ~/S622_robotarm/src
git status
git pull --rebase origin main
git add .
git commit -m "update S622 robot arm source code"
git push
git status
```

## 2.1进入你的 ROS2 src 仓库

```
cd ~/S622_robotarm/src
```

确认当前位置：
```
pwd
```

应该显示：
```
/home/robot/S622_robotarm/src
```

然后确认远程仓库是不是你的 GitHub 仓库：
```
git remote -v
```

正常应该显示：
```
origin  https://github.com/jasonlee0617/S622_robotarm_src.git (fetch)
origin  https://github.com/jasonlee0617/S622_robotarm_src.git (push)
```

## 2.2查看当前代码状态

```
git status
```

### 情况 A：没有任何修改

如果显示：
```
无文件要提交，干净的工作区
```
说明本地没有新修改，不需要上传

### 情况 B：有文件被修改

例如：
```
修改： fairino_planning_core/config/lk_params.yaml修改： yolov8_obb/scripts/yolov8_obb_publisher.py
```
说明这些文件改过，但还没有提交。
### 情况 C：有新文件

例如：
```
未跟踪的文件：    new_launch_file.launch.py
```
说明这个文件还没有被 Git 管理，需要
```
git add
```

## 2.3先拉取 GitHub 最新代码

这是很重要的一步。  
因为如果 GitHub 上有你本地没有的提交，你直接 `git push` 会失败

推荐执行：
```
git pull --rebase origin main
```

如果成功，会看到类似：
```
Successfully rebased and updated refs/heads/main.
```

或者：
```
Already up to date.
```

## 2.4添加你本地修改的代码

如果你想上传所有修改：
```
git add .
```

如果你只想上传某个文件，例如：
```
git add fairino_planning_core/config/lk_params.yaml
```

或者：
```
git add yolov8_obb/scripts/yolov8_obb_publisher.py
```

添加后再看状态：
```
git status
```

## 2.5. 提交代码

提交就是给当前修改保存一个版本
```
git commit -m "update S622 robot arm source code"
```

## 2.6推送到 GitHub

提交完成后执行：
```
git push
```

## 2.7. 最后确认是否同步成功

```
git status
```

如果显示：
```
位于分支 main
您的分支与上游分支 'origin/main' 一致。

无文件要提交，干净的工作区
```