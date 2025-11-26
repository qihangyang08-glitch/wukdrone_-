

我们将从最基础的 **WSL2 环境搭建** 到 **成功编译并运行第一次仿真**。请按照以下步骤严格执行。如果你是 Windows 用户（绝大多数飞控开发者的选择），这是最标准、最顺滑的路径。

### 准备工作：电脑配置
*   系统：Windows 10 或 Windows 11
*   网络：需要能访问 GitHub（如果网络不稳，下载代码会很痛苦，建议准备科学上网工具或配置 GitHub 镜像）。

---

### 第一步：安装 WSL2 (Windows Subsystem for Linux)

我们要在这个 Linux 子系统里编译代码，因为它比 Windows 原生编译快得多，且坑少。

1.  **打开 Windows Powershell（管理员模式）：**
    *   在开始菜单搜 `Powershell`，右键“以管理员身份运行”。
2.  **输入安装命令：**
    ```powershell
    wsl --install
    ```
    *   *注：如果你以前装过 WSL，这里可能会提示已安装。默认会安装 Ubuntu。*
3.  **重启电脑：**
    *   安装完成后，重启电脑。重启后会自动弹出一个黑色窗口（Ubuntu 终端），让你设置 **Username（用户名）** 和 **Password（密码）**。
    *   *记好你的密码，输入时屏幕不会显示星号，这是正常的。*

---

### 第二步：下载 ArduPilot 源码

现在开始，所有操作都在那个 **Ubuntu 终端窗口** 里进行。

1.  **更新系统软件源：**
    ```bash
    sudo apt update
    sudo apt upgrade
    ```
2.  **安装 Git：**
    ```bash
    sudo apt install git
    ```
3.  **克隆源码（这是最关键的一步）：**
    *   ArduPilot 代码库很大，且包含子模块，请耐心等待。
    ```bash
    # 回到主目录
    cd ~
    # 克隆主仓库
    git clone https://github.com/ArduPilot/ardupilot.git
    # 进入目录
    cd ardupilot
    # 更新子模块（这一步最容易因为网络报错，如果失败，多试几次）
    git submodule update --init --recursive
    ```

---

### 第三步：自动配置开发环境

ArduPilot 官方写好了一个脚本，能一键把 gcc 编译器、Python 等环境装好。

1.  **运行安装脚本：**
    ```bash
    Tools/environment_install/install-prereqs-ubuntu.sh -y
    ```
2.  **重新加载配置：**
    *   脚本跑完后，会提示你需要 Log out。你可以直接执行下面这句让环境变量生效：
    ```bash
    . ~/.profile
    ```
    *(注意：`~/.profile` 前面有个点 `.` 和空格)*

---

### 第四步：第一次编译 (Compile)

我们要用 `waf` 工具来构建代码。

1.  **配置飞控板类型：**
    *   我们先编译 **SITL** 版本（即电脑仿真版）。
    ```bash
    ./waf configure --board sitl
    ```
    *   *成功标志：* 最后显示 `'configure' finished successfully`。

2.  **编译多旋翼固件 (Copter)：**
    ```bash
    ./waf copter
    ```
    *   *过程：* 屏幕会疯狂滚动代码编译信息。第一次编译大概需要 2-5 分钟。
    *   *成功标志：* 显示 `Build commands will be stored in build/sitl/compile_commands.json` 且没有红色报错。

---

### 第五步：运行仿真 (Run SITL)

激动人心的时刻来了。

1.  **启动仿真器：**
    ```bash
    sim_vehicle.py -v ArduCopter --console --map
    ```
    *   `-v ArduCopter`: 指定机型为多旋翼。
    *   `--console`: 弹出一个控制台窗口（用于看状态、输指令）。
    *   `--map`: 弹出一个地图窗口（看飞机在哪）。

2.  **观察现象：**
    *   你应该会看到几个新窗口弹出。
    *   在 Console 窗口中，你会看到参数加载，最后显示 `GPS: 3D Fix`，此时飞机处于 **Disarmed**（加锁）状态。

3.  **试飞一下：**
    *   在 `MAVProxy` 的控制台输入以下指令：
    ```bash
    mode guided      # 切换到引导模式
    arm throttle     # 解锁油门
    takeoff 10       # 起飞 10 米
    ```
    *   看地图窗口，飞机图标应该变成了红色，并且高度数值开始上升！

---

### 阶段一“毕业作业”：Hello WuK

既然代码已经在你手里了，我们来做第一点微小的修改，证明你掌握了代码控制权。

**任务：** 修改代码，让仿真启动时，在终端打印一行字：`Wait for WuK System Setup...`

1.  **找到文件：**
    *   在 Ubuntu 终端里，使用 nano 编辑器（或者你在 Windows 下用 VSCode 连接 WSL 打开文件，推荐后者，但现在先用 nano 快速搞定）。
    ```bash
    nano ArduCopter/Copter.cpp
    ```

2.  **修改代码：**
    *   利用 `Ctrl+W` 搜索关键字 `void Copter::setup()`。这是飞控上电初始化的函数。
    *   在 `setup()` 函数的大括号 `{` 进去后的前几行，加上这句代码：
    ```cpp
    // 你的修改
    hal.console->printf("\n\n----------------------------\n");
    hal.console->printf("Wait for WuK System Setup...\n");
    hal.console->printf("----------------------------\n\n");
    ```

3.  **保存退出：**
    *   按 `Ctrl+O` 回车保存，按 `Ctrl+X` 退出。

4.  **重新编译并运行：**
    ```bash
    ./waf copter
    sim_vehicle.py -v ArduCopter
    ```
    *(这次不需要加 --console --map，只看终端输出)*

5.  **检查结果：**
    *   在滚动的启动日志里，你找到那句 `Wait for WuK System Setup...` 了吗？

