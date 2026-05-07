# Docker 保存命令快速参考
# Docker Save Commands Quick Reference

═══════════════════════════════════════════════════════════════════════════

## 🎯 核心概念 | Core Concept

**问题：** Docker容器停止后，所有安装和设置都会丢失
**Problem:** All installations and settings are lost when Docker container stops

**解决：** 将容器保存为镜像（image）
**Solution:** Save container as an image

═══════════════════════════════════════════════════════════════════════════

## 💾 方法1：提交容器为镜像（最常用）| Method 1: Commit Container

### 基本命令：

```bash
# 退出容器
exit

# 提交容器为新镜像
docker commit <container_name> <new_image_name>:<tag>

# 例子：
docker commit ros2_workstation ros2_gesture_control:v1.0
```

### 带说明的提交：

```bash
docker commit \
  -m "安装了robotics_project_ai和所有依赖" \
  -a "你的名字" \
  ros2_workstation \
  ros2_gesture_control:v1.0
```

### 验证新镜像：

```bash
# 查看镜像列表
docker images

# 应该看到：
# REPOSITORY              TAG    IMAGE ID       CREATED         SIZE
# ros2_gesture_control    v1.0   abc123def456   5 seconds ago   3.5GB
```

═══════════════════════════════════════════════════════════════════════════

## 🚀 使用新镜像 | Using the New Image

### 停止并删除旧容器：

```bash
docker stop ros2_workstation
docker rm ros2_workstation
```

### 从新镜像启动容器：

```bash
xhost +local:docker

docker run -it \
  --name ros2_workstation \
  --network host \
  --device=/dev/video0:/dev/video0 \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v /path/to/your/code:/workspace \
  --privileged \
  ros2_gesture_control:v1.0 \
  bash
```

### 验证内容还在：

```bash
# 进入容器后
pip3 list | grep mediapipe
# 应该显示mediapipe已安装 ✅

ros2 pkg list | grep robotics_project_ai
# 应该显示包存在 ✅
```

═══════════════════════════════════════════════════════════════════════════

## 📦 方法2：保存镜像到文件 | Method 2: Save Image to File

### 保存到文件（用于备份或传输）：

```bash
# 保存镜像
docker save -o ~/ros2_gesture_control_v1.0.tar ros2_gesture_control:v1.0

# 压缩（可选）
gzip ~/ros2_gesture_control_v1.0.tar

# 文件大小约：2-4GB（压缩后）
```

### 从文件加载镜像：

```bash
# 在另一台机器或重装系统后
docker load -i ~/ros2_gesture_control_v1.0.tar.gz

# 验证
docker images | grep ros2_gesture_control
```

═══════════════════════════════════════════════════════════════════════════

## 🔄 方法3：导出容器（不推荐）| Method 3: Export Container

### 导出容器：

```bash
docker export ros2_workstation > ~/container_backup.tar
gzip ~/container_backup.tar
```

### 导入容器：

```bash
cat ~/container_backup.tar.gz | gunzip | docker import - ros2_restored:latest
```

**⚠️ 注意：** export/import会丢失元数据，推荐使用commit或save

═══════════════════════════════════════════════════════════════════════════

## ⚡ 完整工作流程 | Complete Workflow

### 第一次设置：

```bash
# 1. 启动原始容器
docker run -it --name ros2_workstation ... original_image bash

# 2. 在容器内安装所有东西
pip3 install --break-system-packages mediapipe opencv-python numpy torch
# ... 创建项目，编译包等

# 3. 测试确认一切正常
ros2 pkg list | grep robotics_project_ai

# 4. 退出容器
exit

# 5. 保存为新镜像
docker commit ros2_workstation ros2_gesture_control:v1.0

# 6. （可选）保存到文件备份
docker save -o ~/ros2_v1.0.tar ros2_gesture_control:v1.0
gzip ~/ros2_v1.0.tar
```

### 以后使用：

```bash
# 直接使用新镜像启动
docker run -it --name ros2_workstation \
  --network host \
  ... \
  ros2_gesture_control:v1.0 \
  bash

# 进入后一切都在，无需重新安装！
```

═══════════════════════════════════════════════════════════════════════════

## 🔍 常用检查命令 | Common Check Commands

### 查看所有镜像：

```bash
docker images

# 或只看特定的
docker images | grep ros2
```

### 查看镜像详细信息：

```bash
docker inspect ros2_gesture_control:v1.0

# 查看创建时间
docker inspect ros2_gesture_control:v1.0 --format='{{.Created}}'

# 查看大小
docker inspect ros2_gesture_control:v1.0 --format='{{.Size}}'
```

### 查看镜像历史：

```bash
docker history ros2_gesture_control:v1.0
```

═══════════════════════════════════════════════════════════════════════════

## 🏷️ 镜像标签管理 | Image Tag Management

### 添加新标签：

```bash
# 给镜像添加多个标签
docker tag ros2_gesture_control:v1.0 ros2_gesture_control:latest
docker tag ros2_gesture_control:v1.0 ros2_gesture_control:backup
```

### 重命名镜像：

```bash
# 创建新标签
docker tag old_name:old_tag new_name:new_tag

# 删除旧标签
docker rmi old_name:old_tag
```

### 删除镜像：

```bash
# 删除镜像（确保没有容器在使用）
docker rmi ros2_gesture_control:v1.0

# 强制删除
docker rmi -f ros2_gesture_control:v1.0
```

═══════════════════════════════════════════════════════════════════════════

## ⚠️ 重要注意事项 | Important Notes

### 1. 什么会被保存？

✅ **会被保存：**
- 已安装的软件包（pip, apt）
- 修改的配置文件
- /root 目录下的文件
- 创建的目录和文件

❌ **不会被保存：**
- 挂载的卷（-v）中的数据
- 运行中的进程
- 临时文件（/tmp）

### 2. 何时提交？

```bash
✅ 在以下情况后提交：
- 安装新软件包
- 编译ROS2包
- 修改重要配置
- 完成项目设置

⚠️ 不要在以下情况提交：
- 容器运行中有错误
- 测试临时更改
- 不确定更改是否正确
```

### 3. 镜像大小管理：

```bash
# 查看镜像大小
docker images

# 清理未使用的镜像
docker image prune

# 清理所有未使用的数据（谨慎！）
docker system prune -a
```

### 4. 版本控制最佳实践：

```bash
# 好的命名：
ros2_gesture_control:v1.0
ros2_gesture_control:v1.1-with-gpu
ros2_gesture_control:2024-05-07

# 不好的命名：
ros2_gesture_control:latest  # 不知道是什么版本
ros2_gesture_control:test    # 太模糊
```

═══════════════════════════════════════════════════════════════════════════

## 📋 检查清单 | Checklist

### 保存容器前：

- [ ] 所有软件都安装完成
- [ ] ROS2包编译成功
- [ ] 测试确认系统工作
- [ ] 退出容器（exit）

### 保存后验证：

- [ ] 镜像出现在 docker images 列表中
- [ ] 镜像大小合理（2-4GB）
- [ ] 能从新镜像启动容器
- [ ] 新容器中所有内容都在

### 备份（可选但推荐）：

- [ ] 保存镜像到文件
- [ ] 压缩文件
- [ ] 复制到安全位置（U盘、云盘）

═══════════════════════════════════════════════════════════════════════════

## 🎯 快速命令总结 | Quick Command Summary

```bash
# 提交容器
docker commit <container> <image>:<tag>

# 保存镜像
docker save -o file.tar <image>:<tag>

# 加载镜像
docker load -i file.tar

# 查看镜像
docker images

# 从镜像启动
docker run -it --name <name> <image>:<tag> bash

# 删除镜像
docker rmi <image>:<tag>
```

═══════════════════════════════════════════════════════════════════════════

## 💡 实际例子 | Practical Example

```bash
# 场景：在Docker中设置完新系统后保存
# Scenario: Saving after setting up new system in Docker

# 1. 在容器内完成所有工作
(inside container) $ pip3 install mediapipe opencv-python
(inside container) $ colcon build --packages-select robotics_project_ai
(inside container) $ exit

# 2. 保存为新镜像
$ docker commit ros2_workstation ros2_gesture_control:v1.0

# 3. 验证
$ docker images | grep ros2_gesture_control
ros2_gesture_control   v1.0   abc123   1 min ago   3.2GB

# 4. 备份到文件（可选）
$ docker save -o ~/backup/ros2_v1.0.tar ros2_gesture_control:v1.0
$ gzip ~/backup/ros2_v1.0.tar

# 5. 以后直接使用
$ docker run -it --network host ... ros2_gesture_control:v1.0 bash

# 完成！以后每次启动都有完整环境！
```

═══════════════════════════════════════════════════════════════════════════

🎉 记住：commit是您的好朋友！每次重要更改后都commit！
   Remember: commit is your friend! Commit after every important change!

═══════════════════════════════════════════════════════════════════════════
