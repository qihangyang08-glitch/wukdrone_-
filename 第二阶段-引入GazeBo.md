

### ⚠️ 重要概念升级：架构变了

引入 Gazebo 后，我们的系统架构会发生一个**根本性**的变化，请务必理解：

*   **刚才（纯 SITL）：**
    *   **大脑：** ArduPilot
    *   **身体（物理计算）：** `SIM_Frame.cpp` (我们刚才改的 C++ 代码)
    *   **画面：** 2D 地图

*   **现在（SITL + Gazebo）：**
    *   **大脑：** ArduPilot (只负责算 PWM)
    *   **身体（物理计算）：** **Gazebo** (它接管了物理运算，`SIM_Frame.cpp` 被旁路了)
    *   **画面：** 3D 窗口

**这意味着：** 为了在 Gazebo 里看到变构，我们需要写一个 **SDF 模型文件**（类似 XML 格式），告诉 Gazebo：“我有 4 个电机臂，它们通过转轴（Joint）连接在机身上，受 Servo 9 控制。”

---

### 第一步：安装 Gazebo (Garden/Harmonic)和各种依赖包

在你的 WSL Ubuntu 终端里，我们需要安装 Gazebo 模拟器和 ArduPilot 的连接插件。

*(注：以下命令适用于 Ubuntu 22.04，如果你是 20.04，部分源可能不同，建议先确认版本 `lsb_release -a`)*

#### 1. 安装 Gazebo Garden (新一代)
```bash
sudo apt-get update
sudo apt-get install lsb-release wget gnupg

# 添加 Gazebo 源
sudo wget https://packages.osrfoundation.org/gazebo.gpg -O /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] http://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null

# 安装
sudo apt-get update
sudo apt-get install gz-garden

# 安装opencv开发包
sudo apt update
sudo apt install libopencv-dev -y

#安装 GStreamer 开发包
sudo apt update
sudo apt install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev gstreamer1.0-plugins-bad gstreamer1.0-libav gstreamer1.0-gl -y

```

#### 2. 安装 ArduPilot-Gazebo 插件
这是连接 ArduPilot 和 Gazebo 的桥梁。

```bash
# 安装依赖
sudo apt update
sudo apt install libgz-sim8-dev rapidjson-dev

# 回到主目录
cd ~
# 下载插件源码
git clone https://github.com/ArduPilot/ardupilot_gazebo.git

# 编译插件
cd ardupilot_gazebo
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j4
```

#### 3. 配置环境路径
我们需要告诉终端插件在哪里。
打开 `~/.bashrc`：
```bash
code ~/.bashrc
```
在文件末尾添加：
```bash
# Gazebo 配置
export GZ_VERSION=harmonic
export GZ_SIM_SYSTEM_PLUGIN_PATH=$HOME/ardupilot_gazebo/build:$GZ_SIM_SYSTEM_PLUGIN_PATH
export GZ_SIM_RESOURCE_PATH=$HOME/ardupilot_gazebo/models:$HOME/ardupilot_gazebo/worlds:$GZ_SIM_RESOURCE_PATH
```
保存并退出，然后刷新：
```bash
source ~/.bashrc
```

---

### 第二步：运行一个官方 Demo

WSL2 虽然支持 GUI，但默认策略非常保守，往往会调用核显（Microsoft Basic Render Driver）而不是你的 NVIDIA 独显。

为了解决**卡顿**和**连接报错**，我为你整理了一套**“强制独显 + 纯净连接”**的完整执行方案。请按顺序执行。

---

#### 第一阶段：优化显卡配置

##### 第一步：检查 WSL 版本（绝大多数人的坑）

虽然你是在 WSL 里，但如果它是 **WSL 1** 而不是 **WSL 2**，它是绝对无法调用 GPU 的。

1.  **回到 Windows PowerShell（管理员）**，输入：
    ```powershell
    wsl --list --verbose
    ```
    *(或者 `wsl -l -v`)*

2.  **检查结果：**
    *   如果 `VERSION` 列显示的是 **1**：
        *   **修复：** 输入 `wsl --set-version Ubuntu 2`（或者你的发行版名字）。等待转换完成。
    *   如果 `VERSION` 列显示的是 **2**：
        *   继续下一步。

---

##### 第二步：安装 WSLg 专用驱动包 (Ubuntu 内部)

Ubuntu 24.04 有时精简了连接 DirectX 12 的中间层库，我们需要手动补全。

回到 **Ubuntu 终端**，执行：

```bash
# 1. 确保系统最新
sudo apt update

# 2. 安装 Mesa 的 D3D12 驱动和 Vulkan 支持
sudo apt install libgl1 libglx-mesa0 libgl1-mesa-dri mesa-vulkan-drivers libgl1-mesa-dev -y

# 3. (关键) 删除可能冲突的旧配置
# 有时候为了强行开启 GPU，乱设环境变量反而会导致回落到 CPU
unset MESA_LOADER_DRIVER_OVERRIDE
unset MESA_D3D12_DEFAULT_ADAPTER_NAME

# 4.回到 Windows，以管理员身份打开 PowerShell。执行强制更新命令：
wsl --update
wsl --shutdown
```

---

##### 第三步：修改环境变量（修正之前的方案）

之前的 `zink` 方案在某些新硬件上可能失效，导致回滚到 `llvmpipe`。我们要换成最原生的 **d3d12** 方案。

1.  打开配置文件：
    ```bash
    code ~/.bashrc
    ```

2.  **找到之前让你加的那几行 GPU 的配置，全部删掉或注释掉。**

3.  **换成下面这组新的配置：**

    ```bash
    # --- WSL2 GPU Fix ---
    # 强制使用 D3D12 (连接 Windows 显卡的核心通道)
    export MESA_LOADER_DRIVER_OVERRIDE=d3d12

    # 强制 Gazebo/OpenGL 走硬件加速
    export LIBGL_ALWAYS_SOFTWARE=0
    ```

4.  保存退出，并刷新：
    ```bash
    source ~/.bashrc
    ```

5.  **重新打开 Ubuntu 终端**。

6.  **再次检查：**
    ```bash
    glxinfo -B
    ```

**成功标准：**
`Device:` 这一行必须包含 **D3D12** (例如 `D3D12 (NVIDIA GeForce RTX 4060...)`)。
只要这一行变了，Gazebo 就会极其丝滑。


---

#### 第二阶段：解决“协议魔法数”报错 (Connection Fix)

报错 `Incorrect protocol magic` 的原因是：SITL 默认尝试用二进制协议连接，而 Gazebo 插件（ardupilot_gazebo）使用的是 JSON 纯文本协议。我们需要强制 SITL 使用 JSON 模式。

##### 1. 启动 Gazebo (服务端)
在第一个终端窗口，启动 Gazebo 和跑道模型：

```bash
# -s 表示以服务模式运行，-r 表示自动开始播放物理
gz sim -v4 -r iris_runway.sdf
```
*(此时应该能看到 3D 窗口，且因为用了独显，应该比之前流畅)*

##### 2. 启动 SITL (客户端 - 纯净 JSON 模式)
在第二个终端窗口，**不要用 `-f gazebo-iris`**（那个参数包容易带旧配置），我们用最原始的指令：

```bash
# --model JSON : 强制使用 JSON 接口
# --rate 400   : 提高仿真频率
sim_vehicle.py -v ArduCopter --model JSON --console --map
```

---

#### 第三阶段：如果还是连不上（参数注入法）

如果执行完第二阶段，两边还是没反应，或者 MAVProxy 报错连接失败，说明端口没对上。我们需要手动告诉 SITL 往哪里发数据。

请**关掉 SITL**，重新用下面这个**带参数注入**的指令启动：

```bash
sim_vehicle.py -v ArduCopter --model JSON --console --map --custom-location=35.363261,138.730795,584,353
```

**启动后，立刻在 MAVProxy 终端里输入以下指令（确保参数正确）：**

```bash
# 告诉飞控：我要用 JSON 接口
param set SIM_JSON_ENABLE 1

# 告诉飞控：Gazebo 的插件在本地 9002 端口等着呢
# 注意：这个端口必须和 Gazebo 插件的一致，默认是 9002
# 也就是 TCP connection
```

**但是，最最稳妥的方法是直接修改默认参数文件。**
ArduPilot 会保存上次的参数。为了彻底修复，请在 SITL 启动后（MAVProxy 界面），执行：

```bash
param set SIM_JSON_ENABLE 1
param set AHRS_EKF_TYPE 3
reboot
```
*(输入 reboot 后，SITL 会重启)*

---

#### 第四阶段：验证飞行

如果一切正常：
1.  **Gazebo 终端：** 不再疯狂刷屏报错。
2.  **MAVProxy 终端：** 显示 `GPS: 3D Fix`。
3.  **操作：**
    ```bash
    mode guided
    arm throttle
    takeoff 5
    ```
4.  **画面：** Gazebo 里的飞机应该平滑升起，且任务管理器里 GPU 占用率应该上升。

---

#### 总结你的“流畅操控”指令集

**环境准备（仅需做一次）：**
1.  修改 `.bashrc` 加入 NVIDIA 环境变量。
2.  `source ~/.bashrc`。

**日常启动流程：**
1.  **终端 A：** `gz sim -v4 -r iris_runway.sdf`
2.  **终端 B：** `sim_vehicle.py -v ArduCopter --model JSON --console --map`
3.  **终端 B (首次)：** `mode guided` -> `arm throttle` -> `takeoff 5`

**请按此执行。如果 `glxinfo -B` 显示已经是 NVIDIA 了但 Gazebo 还是很卡，请告诉我，那可能是 Gazebo 自身的渲染引擎设置问题。**


---

### 第四步：怎么实现“WuK”的 3D 变构？

要实现你的变构，我们需要创建一个自定义的 `.sdf` 文件（模型描述文件）。

这比写 C++ 要繁琐一些，因为涉及到三维坐标定义。我给你一个 **SDF 文件模板** 的思路，你可以先消化一下：

**SDF 文件逻辑 (WuK.sdf)：**

```xml
<model name="wuk_drone">
  <!-- 1. 机身 (Base Link) -->
  <link name="base_link">
     ... 机身的形状 ...
  </link>

  <!-- 2. 左前机臂 (Arm Link) -->
  <link name="arm_front_left">
     ... 机臂形状 ...
  </link>

  <!-- 3. 关键：连接机身和机臂的关节 (Joint) -->
  <joint name="servo_9_joint" type="revolute">
    <parent>base_link</parent>
    <child>arm_front_left</child>
    <axis>
      <xyz>1 0 0</xyz> <!-- 绕 X 轴旋转 -->
      <limit>
        <lower>-1.57</lower> <!-- -90度 -->
        <upper>0</upper>
      </limit>
    </axis>
  </joint>

  <!-- 4. 插件配置：告诉 ArduPilot 控制这个关节 -->
  <plugin filename="ArduPilotPlugin" name="ArduPilotPlugin">
    <!-- 将 Servo 9 映射到 servo_9_joint -->
    <control channel="8"> <!-- channel 8 对应 Servo 9 -->
      <jointName>servo_9_joint</jointName>
      <multiplier>-1.57</multiplier> <!-- PWM 1000~2000 映射到角度 -->
    </control>
  </plugin>
</model>
```

---

### 你的决策时刻

3D 仿真非常酷，但它会增加“建模”的工作量（你需要去写 xml 定义每个零件的位置）。

**你现在的选项：**
1.  **继续用 2D + Console：** 专注于**飞控算法**（C++ 代码）。虽然丑，但验证物理逻辑足够快。
2.  **转战 Gazebo：** 花 1-2 天时间搞定环境和建模，以后都能看到 3D 变构。

**如果你想看 3D，请先执行“第一步”和“第二步”，告诉我你能否看到那个 Iris 飞机飞起来？** 只要环境通了，我帮你写那个 WuK 的 SDF 文件模板。
