

### 第一步：创建模型目录

Gazebo 的模型都有固定的目录结构。我们需要为 WuK 创建一个新家。

在 Ubuntu 终端执行：

```bash
# 1. 进入模型库目录
cd ~/ardupilot_gazebo/models

# 2. 创建 WuK 目录
mkdir wuk_morph
cd wuk_morph
```

---

### 第二步：编写身份证 (model.config)

我们需要一个配置文件告诉 Gazebo：“这是个什么东西”。

1.  **新建文件：**
    ```bash
    code model.config
    ```
2.  **粘贴以下内容：**
    ```xml
    <?xml version="1.0"?>
    <model>
      <name>WuK Morph Drone</name>
      <version>1.0</version>
      <sdf version="1.6">model.sdf</sdf>
      <author>
        <name>WuK Team</name>
        <email>wuk@example.com</email>
      </author>
      <description>
        A morphing quadcopter with inward-tilting arms.
      </description>
    </model>
    ```
3.  **保存关闭。**

---

### 第三步：编写骨架与关节 (model.sdf) —— **核心步骤**

这是重头戏。我为你写好了一个精简版的 SDF，包含：
*   **1个机身**（方块）
*   **4个电机座**（圆柱，连接在机身上）
*   **关键：** 电机座与机身之间有 **Revolute Joint (旋转关节)**。
*   **插件配置：** 告诉 ArduPilot，**通道 9** 控制这 4 个关节转动。

1.  **新建文件：**
    ```bash
    code model.sdf
    ```

2.  **粘贴以下代码（直接覆盖）：**

    ```xml
    <?xml version="1.0" ?>
    <sdf version="1.9">
      <model name="wuk_morph">
        <!-- 1. 机身 (一个白色方块) -->
        <link name="base_link">
          <pose>0 0 0.1 0 0 0</pose>
          <inertial>
            <mass>1.5</mass>
            <inertia><ixx>0.03</ixx><iyy>0.03</iyy><izz>0.06</izz></inertia>
          </inertial>
          <visual name="visual">
            <geometry><box><size>0.4 0.2 0.05</size></box></geometry>
            <material><ambient>1 1 1 1</ambient><diffuse>1 1 1 1</diffuse></material>
          </visual>
          <collision name="collision">
            <geometry><box><size>0.4 0.2 0.05</size></box></geometry>
          </collision>
        </link>

        <!-- 2. 定义四个变构关节和电机 (这里简化演示2个，左右各一组) -->
        
        <!-- === 右侧电机组 (Motor 1 & 4) - 向内倾转 (向左转) === -->
        <link name="arm_right">
          <pose>0 -0.2 0.1 0 0 0</pose> <!-- 在机身右侧 -->
          <inertial><mass>0.2</mass><inertia><ixx>0.001</ixx><iyy>0.001</iyy><izz>0.001</izz></inertia></inertial>
          <visual name="vis">
            <geometry><cylinder><radius>0.02</radius><length>0.3</length></cylinder></geometry>
            <material><ambient>1 0 0 1</ambient><diffuse>1 0 0 1</diffuse></material> <!-- 红色 -->
          </visual>
          <collision name="col"><geometry><cylinder><radius>0.02</radius><length>0.3</length></cylinder></geometry></collision>
        </link>

        <joint name="morph_joint_right" type="revolute">
          <parent>base_link</parent>
          <child>arm_right</child>
          <axis>
            <xyz>1 0 0</xyz> <!-- 绕 X 轴旋转 -->
            <limit><lower>-1.57</lower><upper>0</upper></limit> <!-- 0 到 -90度 -->
          </axis>
        </joint>

        <!-- === 左侧电机组 (Motor 2 & 3) - 向内倾转 (向右转) === -->
        <link name="arm_left">
          <pose>0 0.2 0.1 0 0 0</pose> <!-- 在机身左侧 -->
          <inertial><mass>0.2</mass><inertia><ixx>0.001</ixx><iyy>0.001</iyy><izz>0.001</izz></inertia></inertial>
          <visual name="vis">
            <geometry><cylinder><radius>0.02</radius><length>0.3</length></cylinder></geometry>
            <material><ambient>0 1 0 1</ambient><diffuse>0 1 0 1</diffuse></material> <!-- 绿色 -->
          </visual>
          <collision name="col"><geometry><cylinder><radius>0.02</radius><length>0.3</length></cylinder></geometry></collision>
        </link>

        <joint name="morph_joint_left" type="revolute">
          <parent>base_link</parent>
          <child>arm_left</child>
          <axis>
            <xyz>1 0 0</xyz> <!-- 绕 X 轴旋转 -->
            <limit><lower>0</lower><upper>1.57</upper></limit> <!-- 0 到 90度 -->
          </axis>
        </joint>

        <!-- 3. 这里还需要定义螺旋桨(LiftDragPlugin)，为了简化，我们先只验证关节转动 -->
        <!-- 真正产生升力的插件配置在下面 -->

        <!-- 4. ArduPilot 插件配置 (大脑连接) -->
        <plugin filename="ArduPilotPlugin" name="ArduPilotPlugin">
          <fdm_addr>127.0.0.1</fdm_addr>
          <fdm_port_in>9002</fdm_port_in>
          <connectionTimeoutMaxCount>5</connectionTimeoutMaxCount>
          <lock_step>1</lock_step>

          <!-- 这里的控制通道定义了变构逻辑 -->
          <!-- 通道 9 (索引8) 控制右侧关节: 1000~2000 映射到 0 ~ -1.57弧度 -->
          <control channel="8">
            <jointName>morph_joint_right</jointName>
            <multiplier>-1.57</multiplier> 
            <offset>0</offset>
            <servo_min>1000</servo_min>
            <servo_max>2000</servo_max>
          </control>

          <!-- 通道 9 (索引8) 同时控制左侧关节: 1000~2000 映射到 0 ~ 1.57弧度 -->
          <control channel="8">
            <jointName>morph_joint_left</jointName>
            <multiplier>1.57</multiplier> 
            <offset>0</offset>
            <servo_min>1000</servo_min>
            <servo_max>2000</servo_max>
          </control>
          
          <!-- 电机动力控制 (如果不加这个，飞机飞不起来，这里仅示意前两个电机) -->
          <control channel="0"><type>VELOCITY</type><offset>0</offset><p_gain>0.2</p_gain><i_gain>0</i_gain><d_gain>0</d_gain><cmd_max>2.5</cmd_max><cmd_min>-2.5</cmd_min><jointName>rotor_0_joint</jointName></control>
          <!-- ... 实际飞行还需要完整的螺旋桨定义，这里先测试变构动作 ... -->

        </plugin>
      </model>
    </sdf>
    ```
    *(注：这是一个精简版，省略了螺旋桨的空气动力学定义，专注于让你看到**“红绿机臂转动”**)*

3.  **保存关闭。**

---

### 第四步：将模型路径加入 Gazebo

我们要让 Gazebo 能找到这个新文件夹。

打开 `~/.bashrc`，确保之前加的那行 `GZ_SIM_RESOURCE_PATH` 包含了 `ardupilot_gazebo/models`。如果你是按我之前的文档操作的，这一步应该已经好了。

为了保险，刷新一下：
```bash
source ~/.bashrc
```

---

### 第五步：见证奇迹的时刻

现在，我们不在 SITL 里飞了，我们只测试 **Gazebo 里的机械动作**。

1.  **启动 Gazebo（空白世界）：**
    ```bash
    gz sim -v4 empty.sdf
    ```

2.  **刷出你的模型：**
    *   在右侧的 **Resource Browser (资源浏览器)** 里，找到 `WuK Morph Drone`。
    *   把它拖拽到地图中间。
    *   *你应该看到一个白盒子，左右连着红色和绿色的圆柱。*

3.  **启动 SITL 连接：**
    ```bash
    cd ~/ardupilot
    sim_vehicle.py -v ArduCopter --model JSON --console
    ```

4.  **控制变形：**
    在 MAVProxy 终端里输入：
    ```bash
    servo set 9 1900
    ```

**观察 Gazebo 画面：**
那个红色和绿色的圆柱（机臂），是不是**同时向内转动并竖起来了**？

*   如果转了，恭喜你！你已经打通了 **SITL指令 -> Gazebo关节** 的链路。
*   接下来只需要把螺旋桨加上去，这架飞机就能真的在 Gazebo 里一边飞一边变形了。

**先试这一步，告诉我能不能看到机臂转动？**
