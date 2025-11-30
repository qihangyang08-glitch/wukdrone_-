

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

在你自己造 WuK 的 3D 模型前，我们先跑通官方的四旋翼，看看 3D 效果。

1.  **在一个终端窗口启动 Gazebo：**
    ```bash
    gz sim -v4 -r iris_runway.sdf
    ```
    *   **现象：** 应该会弹出一个 3D 窗口，里面有一条跑道和一架叫 Iris 的四旋翼。
    *   *如果不弹窗：可能是 WSL 的图形驱动问题，Windows 11 通常直接支持，Windows 10 需要装 VcXsrv。你是 Win11 吗？*

2.  **在另一个终端窗口启动 ArduPilot SITL：**
    *(注意参数的变化：-f gazebo-iris)*
    ```bash
    cd ~/ardupilot
    sim_vehicle.py -v ArduCopter -f gazebo-iris --console --map
    ```

3.  **看效果：**
    *   当 SITL 启动后，你会发现 Gazebo 里的飞机螺旋桨开始转动了！
    *   在 MAVProxy 输入 `mode guided`, `arm throttle`, `takeoff 5`。
    *   **你会在 3D 窗口里看到飞机真的飞起来了！**

---
### 第三步：优化方案


#### 1. 为什么报错 `Incorrect protocol magic 0`？（核心问题）

**一句话解释：语言不通。**

*   **现状：**
    *   你的 **Gazebo 插件**（刚才编译的）是最新版的，它讲的是 **“新版二进制语言”**。
    *   你的 **SITL**（通过 `-f gazebo-iris` 启动）默认配置是用 **“旧版 JSON 文本语言”** 发送数据。
*   **结果：** Gazebo 插件收到数据包，一检查开头（Magic Number），发现是 0（或者格式不对），不是它期待的 `18458`，所以它拒绝通信。这就导致了“两边没反应”。

**✅ 解决方法：**
我们需要强迫 SITL 也讲“新版二进制语言”。

1.  **先启动 Gazebo：**
    ```bash
    gz sim -v4 -r iris_runway.sdf
    ```
2.  **再启动 SITL（不带 -f 参数，先手动配参数）：**
    为了避免 `-f gazebo-iris` 自动加载旧参数，我们用默认模式启动，然后手动改。
    ```bash
    sim_vehicle.py -v ArduCopter --console --map
    ```
    *(注意：这时它跑的是你之前改过的那个“鸠占鹊巢”的 WuK 物理模型，但我们先不管它，我们要先连上 Gazebo)*

3.  **在 MAVProxy 终端修改参数：**
    输入以下指令，把 SITL 切换到 Gazebo 模式：
    ```bash
    param set SIM_GZ_ENABLE 1
    param set SIM_GZ_URI "127.0.0.1:9002"
    ```
    *   `SIM_GZ_ENABLE 1`: 开启新版 Gazebo 接口。
    *   `URI`: 指定通信地址（Gazebo 插件默认监听 9002）。

4.  **重启 SITL：**
    按 `Ctrl+C` 关掉 SITL，然后再次运行：
    ```bash
    sim_vehicle.py -v ArduCopter --console --map
    ```

5.  **观察：**
    这时候 Gazebo 里的报错应该会消失，或者变成 `Connected`。你再试着 `mode guided`, `takeoff`，看看 Gazebo 里的飞机动没动。

---

#### 2. 为什么 Gazebo 很卡？（性能优化）

你的电脑是游戏本（有独显），但 WSL2 默认可能在用 CPU 跑图形渲染（llvmpipe），或者 Gazebo 的实时因数（Real Time Factor）太高。

**✅ 优化方案：**

1.  **强制开启 GPU 加速：**
    在 Ubuntu 终端里，执行以下命令后再启动 Gazebo：
    ```bash
    export LIBGL_ALWAYS_SOFTWARE=0
    export MESA_GL_VERSION_OVERRIDE=3.3
    gz sim -v4 -r iris_runway.sdf
    ```
2.  **检查 WSL 显存：**
    Gazebo 极吃显存。在 Windows 的任务管理器 -> 性能 -> GPU 栏，看看当你运行 Gazebo 时，**3D** 占用率高不高？如果很低，说明 WSL 没用上显卡。
    *   *补救：去 NVIDIA 官网下载最新的 Windows 显卡驱动（GeForce Experience），它会自动给 WSL 安装驱动。*
3.  **不要同时开 Map 和 Console：**
    如果实在卡，SITL 启动时去掉地图：
    ```bash
    sim_vehicle.py -v ArduCopter --console
    ```
    *(省掉一个 Map 窗口能省不少资源)*

---

#### 3. 最关键的概念：为什么“变构”失效了？

你问：“我们昨天不是修改了 SITL 的机型了吗？为什么 Gazebo 里还是 Iris，而且不变形？”

这是一个非常重要的架构分界线。请仔细看下图：

*   **阶段一（昨天做的）：内部物理引擎**
    *   **大脑：** ArduPilot
    *   **身体：** `SIM_Frame.cpp` (**C++ 代码**)
    *   **现象：** 只有数据，没有画面（或只有 2D 地图）。
    *   **你的修改：** 你改了 `SIM_Frame.cpp`，所以昨天在 2D 模式下能坠机。

*   **阶段二（现在做的）：外部物理引擎**
    *   **大脑：** ArduPilot
    *   **身体：** **Gazebo (`iris_runway.sdf`)**
    *   **现象：** 有 3D 画面。
    *   **关键点：** 当连接 Gazebo 时，ArduPilot 会**关闭**内部的 `SIM_Frame.cpp`，全权委托 Gazebo 来计算物理。
    *   **结果：** Gazebo 读取的是 `iris_runway.sdf` 文件。这个文件里定义的是“普通的 Iris 四旋翼”。它**完全不知道**你在 C++ 里写的那些变构代码。

**结论：**
要想在 3D 仿真里看到 WuK 变形，你不能改 C++，你必须**改 SDF 文件**。
你需要创建一个 `wuk.sdf`，在里面定义“电机臂是一个关节，受 Servo 9 控制”。


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
