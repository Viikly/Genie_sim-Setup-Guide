# Genie Sim 仿真环境安装与配置指南

> 一份基于真实实践的 Genie Sim Benchmark **v2.0** 安装、配置与排错指南。

## 重要版本说明

本教程详细记录了 **Genie Sim Benchmark v2.0** 的完整安装与实践过程，基于 Isaac Sim 4.5.0。

请注意，截至2026年初，官方已发布 **v3.0** 版本，主要更新如下：
- 升级 Isaac Sim 至 **v5.1.0**
- **正式支持 NVIDIA RTX 50 系列显卡**
- 提供 Genie G2 机器人模型

**对于新用户，尤其是使用 RTX 50 系列显卡的用户，请优先查阅官方最新文档：**
- **v3.0 指南**：https://agibot-world.com/sim-evaluation/docs/#/v3
- **GitHub 仓库**：https://github.com/AgibotTech/genie_sim

本 v2.0 教程仍可作为环境配置、问题排查思路的参考。

## 目录
- [系统要求](#系统要求)
- [第一步：安装NVIDIA驱动](#第一步安装nvidia驱动)
- [第二步：安装与配置Docker](#第二步安装与配置docker)
- [第三步：获取代码与场景资产](#第三步获取代码与场景资产)
- [第四步：构建Docker镜像](#第四步构建docker镜像)
- [第五步：启动仿真容器](#第五步启动仿真容器)
- [第六步：运行仿真任务](#第六步运行仿真任务)
- [扩展使用](#扩展使用)
- [常见问题 (FAQ)](#常见问题-faq)
- [许可证与致谢](#许可证与致谢)

## 系统要求
<a id="系统要求"></a>

| 组件 | 最低要求 | 推荐配置 |
| :--- | :--- | :--- |
| **操作系统** | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |
| **CPU** | Intel Core i7 (第7代) / AMD Ryzen 5 或更高 | Intel Core i7 (第12代) 或更高 |
| **内存** | 32 GB | 64 GB |
| **GPU** | NVIDIA GeForce RTX 3070 (8GB显存) | NVIDIA GeForce RTX 4090D 或更高 |
| **GPU驱动** | 版本 535.129.03+ | 版本 550.120+ |
| **存储** | 50 GB 可用空间的 SSD | 1 TB NVMe SSD |

## 第一步：安装NVIDIA驱动
<a id="第一步安装nvidia驱动"></a>

1.  **检查当前驱动状态**：
    ```bash
    nvidia-smi
    ```
    如果此命令能正确输出显卡信息表格，可跳至下一步。若报错 `NVIDIA-SMI has failed...`，则需继续安装。

2.  **通过系统仓库安装驱动**（以550版本为例）：
    ```bash
    sudo apt update
    sudo apt install nvidia-driver-550
    ```
    **安装完成后必须重启计算机。**

3.  **处理安全启动 (Secure Boot)**：
    - 重启时，如果系统启用了UEFI安全启动，可能会进入**紫色或蓝色的“MOK管理”界面**。
    - **务必在此界面完成密钥注册**：选择 “Enroll MOK” -> “Continue” -> “Yes”，然后输入你在安装过程中设置的密码。
    - **若键盘在界面无响应**，请换用**有线USB键盘**或**笔记本电脑自带键盘**操作。
    - 这是驱动能否成功加载的关键步骤。

4.  **最终验证**：
    ```bash
    nvidia-smi
    ```
    确认命令输出中包含正确的驱动版本和GPU状态。

## 第二步：安装与配置Docker
<a id="第二步安装与配置docker"></a>

1.  **安装Docker Container**：
    ```bash
    sudo apt update
    sudo apt install docker.io
    ```

2.  **将当前用户加入 `docker` 组**，避免每次使用 `sudo`：
    ```bash
    sudo groupadd docker
    sudo usermod -aG docker $USER
    newgrp docker # 刷新当前会话的组权限，或注销后重新登录
    ```

3.  **验证安装**：
    ```bash
    docker run hello-world
    ```
    看到欢迎信息即表示安装成功。

## 第三步：获取代码与场景资产
<a id="第三步获取代码与场景资产"></a>

1.  **克隆 Genie Sim 仓库**：
    ```bash
    git clone https://github.com/AgibotTech/genie_sim.git
    cd genie_sim
    ```

2.  **下载场景资产 (Assets)**：
    - 访问 [GenieSimAssets on HuggingFace](https://huggingface.co/datasets/agibot-world/GenieSimAssets)。
    - 根据页面指引下载完整资产包。
    - **解压到本地目录，例如 `~/assets`**。确保解压后目录结构为 `~/assets/objects/`, `~/assets/scenes/` 等。

## 第四步：构建Docker镜像
<a id="第四步构建docker镜像"></a>

1.  在项目根目录执行构建命令。**此过程耗时较长（约0.5-2小时），需保持网络稳定**。
    ```bash
    docker build -f ./scripts/dockerfile -t registry.agibot.com/genie-sim/open_source:latest .
    ```

2.  **构建问题排查**：

    **网络超时**：若 `git clone` 或 `pip install` 失败，可尝试在Dockerfile中对应命令前添加 `RUN git config --global http.postBuffer 524288000`。

    **彻底重建**：若缓存导致问题，使用 `docker build --no-cache ...`。

## 第五步：启动仿真容器
<a id="第五步启动仿真容器"></a>

1.  **使用脚本启动容器**（确保在 `~/genie_sim` 目录）：
    ```bash
    SIM_ASSETS=~/assets ./scripts/start_gui.sh
    ```
    **成功标志**：
    *   终端输出 `access control disabled...` 和一长串**容器ID**。
    *   **NVIDIA Isaac Sim 的图形化仿真窗口会自动弹出**。

2.  **解决容器命名冲突错误**：
    如果提示 `“The container name \"/genie_sim_benchmark\" is already in use”`，说明旧容器未清理。
    ```bash
    # 强制移除旧容器
    docker rm -f genie_sim_benchmark
    # 重新启动
    SIM_ASSETS=~/assets ./scripts/start_gui.sh
    ```
    **重要**：请保持运行此命令的终端窗口开启，它维持着容器运行。

## 第六步：运行仿真任务
<a id=“第六步运行仿真任务”></a>

Genie Sim 采用 **客户端-服务器架构**，需要两个终端协作。

### 1. 启动 gRPC 服务器
*   **打开一个新终端**，进入项目目录：
    ```bash
    cd ~/genie_sim
    ```
*   进入正在运行的容器内部：
    ```bash
    ./scripts/into.sh
    ```
    提示符变为 `root@[容器ID]:~/workspace/main#` 表示成功。
*   在容器内启动服务器：
    ```bash
    run_server --enable_curobo True
    ```
    成功启动后，终端将开始持续滚动日志。

### 2. 运行基准测试任务
*   **再打开一个新终端**（在宿主机，无需进入容器）：
    ```bash
    cd ~/genie_sim
    ./scripts/autorun.sh genie_task_supermarket_stock_shelf
    ```
*   如果一切顺利，客户端将连接到服务器，仿真任务将在 Isaac Sim 图形窗口中开始运行。

**至此，你已成功运行第一个仿真任务！**

## 扩展使用
<a id="扩展使用"></a>

掌握基础运行后，可通过 `./scripts/autorun.sh` 探索更多功能模式：

| 模式 | 命令示例 | 说明 |
| :--- | :--- | :--- |
| **基准测试** | `./scripts/autorun.sh genie_task_cafe_espresso` | 运行其他评测任务。 |
| **键盘遥控** | `./scripts/autorun.sh genie_task_cafe_espresso keyboard` | 用键盘实时控制机器人（需先启动 `run_server`）。 |
| **运行模型** | `./scripts/autorun.sh iros_open_drawer infer UniVLA` | 运行基线AI模型（需先准备模型文件）。 |
| **清理** | `./scripts/autorun.sh clean` | 清理临时文件与进程。 |

## 常见问题 (FAQ)
<a id="常见问题-faq"></a>

<details>
<summary><b>Q1: 运行 `nvidia-smi` 始终失败，提示“Driver not loaded”？</b></summary>

这是典型的 **安全启动 (Secure Boot) 密钥未注册** 问题。
*   **解决方案**：重启系统，在UEFI引导阶段出现的**蓝色“Mok Management”界面**中完成注册。
*   **如果界面不出现或键盘无效**：可尝试进入BIOS设置，**暂时禁用Secure Boot**，进入系统验证驱动正常后，再重新启用Secure Boot。

</details>

<details>
<summary><b>Q2: 任务运行时提示“Failed to connect to gRPC server[0]”？</b></summary>

这表示 **gRPC服务器未启动**。
*   **解决方案**：确保你已严格按照教程，在**另一个独立的终端**中成功执行了 `run_server` 命令。这是最常见的疏忽。

</details>

<details>
<summary><b>Q3: 报错找不到 `object_parameters.json` 等资产文件？</b></summary>

*   **检查资产路径**：确认 `SIM_ASSETS` 环境变量指向的路径（如 `~/assets`）完全正确。
*   **检查解压结果**：确认资产包已**完整解压**，目录结构应为 `~/assets/objects/benchmark/...`。

</details>

<details>
<summary><b>Q4: Isaac Sim 窗口卡顿或无响应正常吗？</b></summary>

**首次启动或加载新场景时**，由于需要编译着色器和加载资源，界面出现几分钟的卡顿是**正常现象**。请观察运行 `run_server` 的终端，只要日志在持续滚动，程序就在正常工作。

</details>

<details>
<summary><b>Q5: Docker容器启动失败，如何排查？</b></summary>

*   使用 `docker ps -a` 查看容器状态。
*   使用 `docker logs <容器ID>` 查看具体错误日志。
*   确保宿主机的显卡驱动和Docker配置（尤其是NVIDIA Container Toolkit）正确。

</details>

## 许可证与致谢
<a id="许可证与致谢"></a>

*   **本教程** (`genie-sim-setup-guide`) 采用 [MIT License](LICENSE) 授权。
*   本教程所描述和引用的核心仿真框架 **[Genie Sim Benchmark](https://github.com/AgibotTech/genie_sim)**，其源代码与数据遵循其自身的开源许可证：
    *   `source/geniesim` 与 `source/data_collection` 目录下的内容基于 **Mozilla Public License 2.0 (MPL-2.0)**。
    *   `source/scene_reconstruction` 等项目包含多重许可证，具体细节请参阅其项目内的 LICENSE 文件。
*   本教程为基于 Genie Sim v2.0 的实践记录，与官方无隶属关系。官方版本如有更新，请以其 [用户指南](https://agibot-world.com/sim-evaluation/docs/#/v2) 和 [GitHub仓库](https://github.com/AgibotTech/genie_sim) 为准。
