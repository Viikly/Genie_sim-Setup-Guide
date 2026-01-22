# Genie_sim-Setup-Guide
通过 Docker 容器方式成功安装和运行 Genie Sim Benchmark v2.0 的完整步骤与问题排查指南。包含对 NVIDIA 驱动、安全启动及仿真任务连接错误的重点解决思路。
Genie Sim 机器人仿真环境安装与配置指南

重要版本说明

本教程旨在为你提供一个清晰的安装路线图。在开始操作前，请务必确认你使用的版本：

Genie Sim v3.0 (2026年1月发布)

核心更新：Isaac Sim 升级至 v5.1.0、正式支持RTX 50系列显卡、提供G2机器人模型、支持3D高斯泼溅场景重建等。

官方指南：v3.0 用户指南

适用情况：如果你使用的是RTX 50系列（如RTX 5090）或更新的NVIDIA显卡，必须使用v3.0版本。 使用旧版可能导致兼容性问题。

Genie Sim v2.0 (本教程实践基础)

历史实践：本教程的详细步骤和问题解决方案基于v2.0版本的长期实践。

官方指南：v2.0 用户指南（将上述链接中的 v3 改为 v2）

适用情况：适用于RTX 30/40系列等显卡。如果你在安装v3.0时遇到困难，可参考本教程中关于系统配置、驱动和Docker的通用方法进行排查。

行动建议：

访问上述官方指南链接，这是最权威的安装依据。

将本教程作为实践补充与问题解决手册。许多底层配置（如Linux驱动、Docker）和典型错误（如容器冲突、资产路径）的解决思路是通用的。

关注官方GitHub仓库的 Releases 页面，以获取最新的版本公告和注意事项。

前置条件与准备工作
系统要求
无论哪个版本，都需要满足以下硬件基础。v3.0对显卡和Isaac Sim版本有更高要求。

组件	v2.0 最低要求	v3.0 关键变化与建议
操作系统	Ubuntu 22.04 LTS	推荐 Ubuntu 22.04
GPU	NVIDIA RTX 3070 (8GB) 或更高	必须为NVIDIA显卡。RTX 50系列仅支持v3.0。
GPU驱动	版本 535+	需匹配CUDA 12.4+及Isaac Sim 5.1.0要求，建议安装最新驱动。
Isaac Sim	4.5.0	5.1.0 (随v3.0 Docker镜像提供)
内存/存储	32GB RAM / 50GB SSD	建议64GB RAM / 1TB NVMe SSD
第一步：安装NVIDIA显卡驱动（通用步骤）
这是仿真运行的基础，步骤通用。

禁用开源驱动：编辑 /etc/modprobe.d/blacklist-nouveau.conf，添加：

text
blacklist nouveau
options nouveau modeset=0
然后更新并重启：sudo update-initramfs -u && sudo reboot

安装官方驱动：从 NVIDIA官网 下载或使用系统仓库安装。例如安装550版本：

bash
sudo apt update
sudo apt install nvidia-driver-550
sudo reboot
处理安全启动：重启时若出现蓝色“Mok Management”界面，必须完成密钥注册。如果键盘无响应，请换用有线USB键盘或笔记本自带键盘。如果反复失败，可考虑进入BIOS暂时禁用Secure Boot，进入系统后再重新启用。

验证安装：

bash
nvidia-smi
确认命令成功输出显卡信息，且CUDA版本符合预期（v3.0需12.4+）。

第二步：安装与配置Docker（通用步骤）
bash
# 1. 安装Docker
sudo apt update
sudo apt install docker.io
# 2. 将用户加入docker组，避免每次使用sudo
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker # 或注销重新登录生效
# 3. 验证安装
docker run hello-world
获取与运行Genie Sim容器
从这里开始，v2.0与v3.0的操作路径可能出现差异，请以你的目标版本官方指南为准。

第三步：获取代码与资产
克隆仓库：

bash
git clone https://github.com/AgibotTech/genie_sim.git
cd genie_sim
# 如需特定版本，请使用 git checkout 切换到对应标签，如 `git checkout v3.0.0`
下载场景资产：

访问 GenieSimAssets on HuggingFace。

下载并解压到本地目录，如 ~/assets。

关键：确保解压后 ~/assets 目录下直接包含 objects, scenes 等文件夹。

第四步：构建与启动容器
核心区别：v3.0的Docker镜像基于Isaac Sim 5.1.0重新构建，支持新硬件和G2机器人。构建命令可能更新，请优先查看仓库 README.md。

构建Docker镜像 (此过程耗时较长)：

bash
# 请务必确认当前目录下的 `scripts/dockerfile` 是否为对应版本所需文件
docker build -f ./scripts/dockerfile -t registry.agibot.com/genie-sim/open_source:latest .
构建问题排查：

网络问题：Dockerfile中的 git clone 或 pip install 可能因网络超时失败。尝试在Dockerfile的 RUN git clone 命令前添加 RUN git config --global http.postBuffer 524288000。

完全重建：若缓存导致异常，使用 docker build --no-cache ...。

启动仿真环境：

bash
# 将 ~/assets 替换为你的实际资产路径
SIM_ASSETS=~/assets ./scripts/start_gui.sh
成功标志：

终端输出 access control disabled... 和一串容器ID。

Isaac Sim图形窗口应自动弹出。

解决容器冲突：
如果提示容器名 “/genie_sim_benchmark” 已存在，说明旧容器未清理：

bash
docker rm -f genie_sim_benchmark
# 重新执行启动命令
SIM_ASSETS=~/assets ./scripts/start_gui.sh
保持运行此命令的终端窗口开启，它维持着容器运行。

运行仿真任务 (核心工作流不变)
无论v2.0还是v3.0，“客户端-服务器”架构和基本工作流保持一致。

第五步：启动gRPC服务器
打开新终端，进入项目目录：

bash
cd ~/genie_sim
进入已运行的容器：

bash
./scripts/into.sh
提示符变为 root@[容器ID]:/workspace/main# 表示成功。

在容器内启动服务器：

bash
run_server --enable_curobo True
成功则终端开始滚动日志并等待连接。

第六步：运行任务
再打开一个新终端（在宿主机，非容器内）：

bash
cd ~/genie_sim
# 运行一个基准测试任务，任务名称请参考官方文档
./scripts/autorun.sh genie_task_supermarket_stock_shelf
如果一切顺利，客户端将连接到服务器，仿真任务在Isaac Sim窗口中开始运行。

常见问题与通用解决方案 (FAQs)
以下问题在v2.0/v3.0中均可能出现。

Q1: nvidia-smi 工作正常，但仿真窗口无法启动或黑屏？
A1: 首先检查Docker容器的显示配置是否正确挂载了 DISPLAY 环境变量和 /tmp/.X11-unix。确保宿主机已安装并运行了X11服务。可尝试在运行 start_gui.sh 前执行 xhost +local:（注意安全风险）。

Q2: 任务运行时提示 “Failed to connect to gRPC server[0]”?
A2: 这是最常见问题，意味着服务器未启动。请严格按照上述步骤，确保已在另一个独立终端中成功运行了 run_server 命令。

Q3: 报错找不到 object_parameters.json 等资产文件？
A3: 请再次确认 SIM_ASSETS 环境变量指向的路径完全正确，并且该路径下的资产文件已完整解压，目录结构与官方资产包一致。

Q4: Isaac Sim 窗口卡顿或无响应？
A4: 首次启动或加载新场景时，由于需要编译着色器和加载资源，出现几分钟的卡顿是正常现象。请观察运行 run_server 的终端，只要日志在持续滚动，程序就在工作。

Q5: 如何更新到新版本？
A5: 1. 拉取最新的项目代码 (git pull)。2. 建议删除旧的Docker镜像和容器 (docker rm -f genie_sim_benchmark; docker rmi ...)。3. 按照新版本的 README 或指南，重新构建Docker镜像并启动。

总结
版本第一：始终以 官方用户指南 和 GitHub仓库的 README 为最高优先级。

原理通用：系统驱动、Docker配置、客户端-服务器架构的理解和常见错误排查思路，在不同版本间是通用的。

动手实践：仿真的配置是一个系统工程，遇到问题时，仔细阅读错误信息、回溯操作步骤、利用网络搜索和官方Issues是解决问题的关键能力。

祝你顺利搭建仿真环境，开启机器人学习与开发之旅！
