

# 🚀 第二阶段：构建 WuK 专用仿真模型 (SITL Backend)

**核心目标：** 让 SITL 仿真器“理解”你的变构结构。
原生的 SITL 认为所有四旋翼都是电机固定的。你需要写一段 C++ 代码，告诉物理引擎：**“我的电机是可以转动的，且转动角度受第 5 个伺服电机（Servo）控制。”**

我们不用 Gazebo 那么重的 3D 引擎，先在 **ArduPilot 内置的轻量级物理引擎 (SIM_Frame)** 里实现。

### 任务 1：定位并理解物理后端

请在 VS Code 中打开文件：
`libraries/SITL/SIM_Frame.cpp`

**阅读代码逻辑（约 10 分钟）：**
1.  搜索 `Frame::setup_frame` 函数。
2.  你会看到一个巨大的 `switch` 语句，根据 `frame_type`（X型、+型、H型...）来配置电机位置。
3.  找到 `case FrameType::QUAD:` (标准四旋翼)。
    *   你会看到类似 `motors[0] = Motor(90, ...)` 的代码。这定义了电机的角度和位置。

### 任务 2：创建 WuK 的物理类

我们要“继承”或“修改”现有的物理模型。为了不破坏原有代码，我们**在 `SIM_Frame.cpp` 的末尾添加一种新的机型逻辑**。

**操作步骤：**

1.  **定义机型 ID：**
    虽然正规做法是去头文件加 enum，但为了快速验证，我们直接复用一个不常用的机型，比如 `FrameType::PLUS`（十字型），或者直接硬改 `QUAD` 的逻辑。
    *建议：直接修改 `QUAD` 下的逻辑，因为默认就是 Quad。*

2.  **实现动态电机倾转逻辑：**
    SITL 的物理计算核心在 `Frame::update_forces` 函数里。这里计算了每帧的受力。

    请在 `SIM_Frame.cpp` 中搜索 `void Frame::update_forces`。

3.  **编写变构代码（Hardcore 环节）：**

    在 `update_forces` 函数的开头，你需要获取“变构舵机”的输出值，并实时修改电机的推力矢量。

    **请将以下逻辑翻译成 C++ 代码插入到 `update_forces` 函数的最前面：**

    ```cpp
    // 假设我们用 舵机通道 5 (Servo 5) 控制变形
    // input.servos[] 数组存储了飞控输出的 PWM 值 (1000~2000)
    
    // 1. 获取变形进度 (0.0 = 垂直/飞行, 1.0 = 水平/地面)
    // 假设 PWM 1100 是飞行，1900 是地面
    float morph_pwm = input.servos[4]; // 数组下标从0开始，所以通道5是4
    float morph_ratio = (morph_pwm - 1100.0f) / (1900.0f - 1100.0f);
    
    // 限制范围在 0.0 到 1.0
    if (morph_ratio < 0.0f) morph_ratio = 0.0f;
    if (morph_ratio > 1.0f) morph_ratio = 1.0f;
    
    // 2. 计算当前的物理倾角 (比如最大 90度)
    float tilt_angle_rad = radians(morph_ratio * 90.0f);
    
    // 3. 动态修改电机推力方向
    // ArduPilot 的 SITL 并没有直接暴露 "set_motor_angle" 的接口，
    // 我们通常通过 Hack 方式：在计算合力时，手动旋转推力矢量。
    
    // 这是一个简化版的 Hack 思路，先不做复杂的旋转矩阵，先做现象验证
    // 我们可以打印这个 ratio，看看你在 Mavproxy 里动通道 5 时，这里变没变。
    if (hal.scheduler->millis() % 1000 == 0) {
        printf("Current Morph Ratio: %f \n", morph_ratio);
    }
    ```

### 任务 3：验证链路

作为 CS 学生，你懂的：**写代码 -> 编译 -> 验证数据流**。

1.  **修改代码：** 把上面那段 `printf` 逻辑加到 `libraries/SITL/SIM_Frame.cpp` 的 `update_forces` 函数第一行。
2.  **编译：** VS Code 终端 -> `./waf copter`。
3.  **运行：** `sim_vehicle.py -v ArduCopter`。
4.  **测试：**
    *   在黑窗口（MAVProxy）输入 `servo set 5 1900`（模拟输出 PWM）。
    *   观察终端里打印的 `Current Morph Ratio` 变了没？
    *   输入 `servo set 5 1100`，看变回去没。

