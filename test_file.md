# WuK SITL 测试完整流程

## 一、编译

### 1.1 配置编译环境
```bash
cd ~/ardupilot
./waf configure --board sitl
```

**期望输出**:
```
Setting board to                         : sitl
Using toolchain                          : native
Checking for 'g++' (C++ compiler)        : /usr/lib/ccache/g++
Checking for 'gcc' (C compiler)          : /usr/lib/ccache/gcc
...
'configure' finished successfully (3.7s)
```

### 1.2 编译ArduCopter
```bash
./waf copter
```

**期望输出**:
```
Waf: Entering directory `/home/qwssx2/ardupilot/build/sitl'
[1338/1398] Compiling ArduCopter/mode_morph.cpp
[1392/1398] Compiling ArduCopter/Copter.cpp
[ 922/1398] Compiling libraries/SITL/SIM_Frame.cpp
[1398/1398] Linking build/sitl/bin/arducopter
Waf: Leaving directory `/home/qwssx2/ardupilot/build/sitl'
'build' finished successfully (XX.XXXs)
```

### 1.3 验证编译产物
```bash
ls -lh build/sitl/bin/arducopter
```

**期望输出**:
```
-rwxr-xr-x 1 qwssx2 qwssx2 4.3M Dec 10 22:53 build/sitl/bin/arducopter
```

---

## 二、启动SITL

### 2.1 方法1: 使用快速启动脚本
```bash
cd ~/ardupilot
./test_wuk_sitl.sh
```

**期望输出**:
```
=========================================
WuK SITL Testing Script
=========================================

[1/2] Starting SITL with WuK frame...
SITL PID: 12345
等待SITL启动完成...

[2/2] SITL已启动
MAVProxy commands:
  mode LOITER  # 切换到LOITER
  arm throttle # 解锁
  rc 3 1600    # 起飞
  rc 7 1900    # 测试RC7(UART消息)
  rc 9 1900    # 触发变形

按 Ctrl+C 停止SITL
=========================================
```

同时会打开MAVProxy控制台和地图窗口。

### 2.2 方法2: 手动启动(推荐用于调试)
```bash
cd ~/ardupilot
Tools/autotest/sim_vehicle.py -v ArduCopter -f wuk --console --map \
    --add-param-file=Tools/autotest/default_params/copter-wuk.parm
```

**期望输出**:
```
Using waf configure options: --board sitl
Using existing apj_tool.py
WAF build configured
[INFO] Using frame: wuk (4 motors)
Starting SITL...
Starting MAVProxy...
Loaded module console
Loaded module map
MAVProxy master> 
```

---

## 三、基础功能测试

### 3.1 检查当前模式
在MAVProxy控制台输入:
```
status
```

**期望输出**:
```
Status: EKF2 IMU0: in-flight
        Mode: LOITER
        Lat: -35.3632621
        Lng: 149.1652374
        Alt: 584
```

如果不是LOITER,手动切换:
```
mode LOITER
```

**期望输出**:
```
Got COMMAND_ACK: COMMAND_LONG: ACCEPTED
Mode LOITER
```

### 3.2 解锁测试
```
arm throttle
```

**期望输出**:
```
Got COMMAND_ACK: COMMAND_LONG: ACCEPTED
ARMED
```

### 3.3 起飞测试
```
rc 3 1600
```

**期望行为**:
- 飞机开始上升
- 地图上高度数值逐渐增加
- 等待高度稳定在5米左右

**查看高度**:
```
status
```

**期望输出** (稳定后):
```
Alt: 589  (即相对地面约5米)
```

悬停:
```
rc 3 1500
```

---

## 四、UART消息监控测试

### 4.1 测试RC7通道
```
rc 7 1100
```

**期望在MAVProxy控制台看到**:
```
WuK RC7: 1100 -> ACT1
```

继续测试不同值:
```
rc 7 1500
rc 7 1900
```

**每次应看到**:
```
WuK RC7: 1500 -> ACT1
WuK RC7: 1900 -> ACT1
```

### 4.2 测试RC8通道
```
rc 8 1100
rc 8 1500
rc 8 1900
```

**期望每次看到**:
```
WuK RC8: 1100 -> ACT2
WuK RC8: 1500 -> ACT2
WuK RC8: 1900 -> ACT2
```

**✅ 验证点**: 
- RC7/8改变时立即显示UART命令消息
- 消息格式正确,包含通道号和PWM值

---

## 五、变形功能测试

### 5.1 触发变形
```
rc 9 1900
```

**期望立即看到**:
```
Mode MORPH
Got COMMAND_ACK: COMMAND_LONG: ACCEPTED
```

### 5.2 观察SITL终端输出
在启动SITL的终端窗口中,应该每隔100ms看到:

```
SITL WuK: FC angle=0.0° -> PWM=1000 (morph active)
SITL WuK: FC angle=1.5° -> PWM=1017 (morph active)
SITL WuK: FC angle=3.0° -> PWM=1033 (morph active)
...
SITL WuK: FC angle=43.5° -> PWM=1483 (morph active)
SITL WuK: FC angle=45.0° -> PWM=1500 (morph active)
SITL WuK: FC angle=45.0° -> PWM=1500 (morph active)  [停留约2秒]
SITL WuK: FC angle=45.0° -> PWM=1500 (morph active)
SITL WuK: FC angle=46.5° -> PWM=1517 (morph active)
...
SITL WuK: FC angle=88.5° -> PWM=1983 (morph active)
SITL WuK: FC angle=90.0° -> PWM=2000 (morph active)
```

### 5.3 变形完成
约5-6秒后,应看到:
```
Mode GROUND
Got COMMAND_ACK: COMMAND_LONG: ACCEPTED
```

**✅ 验证点**:
1. ✅ 角度从0°逐渐增加到45° (约3秒, 15°/s)
2. ✅ 在45°停留约2秒
3. ✅ 角度从45°继续增加到90° (约3秒)
4. ✅ 到达90°后自动切换到GROUND模式
5. ✅ SITL终端持续显示FC angle变化

### 5.4 验证SITL读取FC命令
**关键特征**:
- 显示 `FC angle=X.X°` 而不是 `rc9=XXXX`
- 显示 `(morph active)` 表示FC在MORPH模式
- PWM值 = 1000 + (angle/90)*1000

**计算验证**:
- 0° → PWM=1000 ✓
- 45° → PWM=1500 ✓
- 90° → PWM=2000 ✓

### 5.5 退出变形模式
```
rc 9 1100
```

**期望看到**:
```
Mode LOITER
Got COMMAND_ACK: COMMAND_LONG: ACCEPTED
```

---

## 六、日志分析

### 6.1 查看生成的日志文件
```bash
ls -lh wuk_gb.log wuk_gb.jsonl
```

**期望输出**:
```
-rw-r--r-- 1 qwssx2 qwssx2 125K Dec 10 23:15 wuk_gb.log
-rw-r--r-- 1 qwssx2 qwssx2 456K Dec 10 23:15 wuk_gb.jsonl
```

### 6.2 检查文本日志
```bash
head -20 wuk_gb.log
```

**期望输出**:
```
WUK_LOG: time_us=1234567890 angle=0.0 target_pwm=1000 dt=0.001000 roll_deg=0.123 pitch_deg=-0.456 yaw_deg=89.234
 motor1 applied_pwm=1000 cmd=0.000 cur=0.000 volt=12.600 thrust_N=0.000 vec=(0.000,0.000,-0.000)
 motor2 applied_pwm=1000 cmd=0.000 cur=0.000 volt=12.600 thrust_N=0.000 vec=(0.000,0.000,-0.000)
...
```

**✅ 验证点**:
- 包含 `angle=X.X` 字段 (不是rc9)
- applied_pwm从1000逐渐变化到2000
- thrust_N和vec=(x,y,z)记录推力向量

### 6.3 检查JSON日志
```bash
head -3 wuk_gb.jsonl | python3 -m json.tool
```

**期望输出**:
```json
{
  "time_us": 1234567890,
  "elapsed_s": 0.123456,
  "dt": 0.001000,
  "fc_angle": 0.0,
  "fc_morph_active": true,
  "target_pwm": 1000,
  "per_motor": {
    "applied_pwm": [1000, 1000, 1000, 1000],
    "cmd": [0.0, 0.0, 0.0, 0.0],
    "thrust_N": [0.0, 0.0, 0.0, 0.0]
  }
}
```

**✅ 验证点**:
- ✅ 有 `fc_angle` 字段 (不是rc9)
- ✅ 有 `fc_morph_active` 布尔字段
- ✅ JSON格式正确可解析

### 6.4 过滤变形阶段数据
```bash
grep "angle=" wuk_gb.log | awk '{print $2, $3}' | head -20
```

**期望输出**:
```
angle=0.0 target_pwm=1000
angle=1.5 target_pwm=1017
angle=3.0 target_pwm=1033
...
angle=43.5 target_pwm=1483
angle=45.0 target_pwm=1500
angle=45.0 target_pwm=1500  [重复多次]
angle=46.5 target_pwm=1517
...
angle=90.0 target_pwm=2000
```

### 6.5 统计变形时间
```bash
# 提取变形开始和结束的时间戳
grep "angle=0.0" wuk_gb.log | head -1 | awk '{print $2}'
grep "angle=90.0" wuk_gb.log | tail -1 | awk '{print $2}'
```

**手动计算**:
```
结束时间 - 开始时间 ≈ 5-6秒
```

---

## 七、数据可视化 (可选)

### 7.1 编译日志解析器
```bash
cd tools
g++ -o wuk_log_parser wuk_log_parser.cpp
cd ..
```

### 7.2 解析日志
```bash
./tools/wuk_log_parser wuk_gb.log
```

**期望输出**:
```
Parsing wuk_gb.log...
Total frames: 5432
Time span: 5.432 seconds
Average dt: 0.001 seconds
Morph angle range: 0.0° to 90.0°
Parsed data written to wuk_parsed.csv
```

### 7.3 生成图表
```bash
python3 tools/wuk_plot.py
```

**期望输出**:
```
Plotting wuk_parsed.csv...
Generated: wuk_attitude.png
Generated: wuk_thrust_delta.png
Generated: wuk_tilt.png
Generated: wuk_total_vertical.png
```

### 7.4 查看图表
```bash
ls -lh wuk_*.png
```

**期望看到4个图像文件**:
- `wuk_attitude.png` - Roll/Pitch/Yaw vs 时间
- `wuk_thrust_delta.png` - 推力增量 vs 时间
- `wuk_tilt.png` - 倾转角度 vs 时间
- `wuk_total_vertical.png` - 垂直推力 vs 时间

---

## 八、自动化测试 (可选)

### 8.1 准备
确保SITL已启动(方法2.1或2.2)。

### 8.2 运行Python测试脚本
在另一个终端:
```bash
cd ~/ardupilot
python3 Tools/autotest/sitl_wuk_test.py
```

**期望输出**:
```
Connecting to SITL at tcp:127.0.0.1:5760...
Connected!
Waiting for heartbeat...
Heartbeat received from system 1

Setting mode to LOITER...
Mode set successfully

Setting RC channel 7 to 1900...
RC7 set successfully

Setting RC channel 9 to 1900...
RC9 set successfully
Waiting for mode change to MORPH...
Mode changed to MORPH!

Monitoring messages for 10 seconds...
[STATUSTEXT] WuK RC7: 1900 -> ACT1
[STATUSTEXT] WuK RC9: 1900 -> MORPH trigger
...

Test completed successfully!
```

---

## 九、问题排查

### 9.1 SITL启动失败
**症状**: `Unknown frame type: wuk`

**检查**:
```bash
grep -n "wuk" libraries/SITL/SIM_Frame.cpp | head -3
```

**期望输出**:
```
48:static Motor wuk_motors[] =
410:    Frame("wuk", 4, wuk_motors),
```

**解决**: 如果没有,说明代码未正确应用,重新编译。

### 9.2 RC9不触发变形
**症状**: `rc 9 1900` 后没有模式变化

**检查**:
```bash
# 在MAVProxy中
param show RC9
```

**期望看到**: RC9相关参数

**调试**: 在Copter.cpp的process_wuk_switches()中添加打印:
```cpp
hal.console->printf("RC9=%d\n", pwm9);
```

重新编译并测试。

### 9.3 SITL无角度输出
**症状**: SITL终端没有 `SITL WuK: FC angle=...`

**检查1**: 是否在MORPH模式
```
status
```

**检查2**: 查看全局变量是否链接成功
```bash
nm build/sitl/bin/arducopter | grep g_wuk
```

**期望输出**:
```
00000000012345678 B g_wuk_morph_active
00000000012345679 B g_wuk_target_angle
00000000012345680 B g_wuk_last_update_ms
```

### 9.4 消息不显示
**症状**: RC7/8改变但没有STATUSTEXT消息

**检查**: MAVProxy的消息过滤
```
set moddebug 3
```

或查看日志:
```bash
ls -lh logs/
mavlogdump.py --types STATUSTEXT logs/00000001.BIN
```

---

## 十、测试检查清单

### 编译阶段
- [ ] `./waf configure --board sitl` 成功
- [ ] `./waf copter` 编译无错误
- [ ] `build/sitl/bin/arducopter` 存在且约4.3MB

### 启动阶段
- [ ] SITL启动成功,显示 `Using frame: wuk`
- [ ] MAVProxy控制台打开
- [ ] 地图窗口显示
- [ ] 当前模式为LOITER (或可手动切换)

### 基础功能
- [ ] `arm throttle` 解锁成功
- [ ] `rc 3 1600` 起飞成功
- [ ] 高度稳定在5米左右

### UART消息
- [ ] `rc 7 1900` 显示 `WuK RC7: 1900 -> ACT1`
- [ ] `rc 8 1500` 显示 `WuK RC8: 1500 -> ACT2`
- [ ] 不同值都能触发消息

### 变形功能
- [ ] `rc 9 1900` 触发模式切换到MORPH
- [ ] SITL终端显示 `SITL WuK: FC angle=...`
- [ ] 角度从0°逐渐增加到45° (约3秒)
- [ ] 在45°停留约2秒
- [ ] 角度从45°增加到90° (约3秒)
- [ ] 到达90°自动切换到GROUND模式
- [ ] PWM值从1000变化到2000

### 日志记录
- [ ] `wuk_gb.log` 生成且包含 `angle=` 字段
- [ ] `wuk_gb.jsonl` 生成且JSON格式正确
- [ ] 日志包含 `fc_angle` 和 `fc_morph_active` 字段
- [ ] 日志记录完整变形过程

### 数据分析
- [ ] 日志解析器编译成功
- [ ] 生成CSV数据文件
- [ ] 生成4张图表
- [ ] 图表显示合理的数据趋势

---

## 十一、完整测试脚本

### 11.1 一键测试脚本
保存为 `full_test.sh`:
```bash
#!/bin/bash
# WuK SITL完整测试脚本

set -e
cd ~/ardupilot

echo "========================================="
echo "WuK SITL 完整测试流程"
echo "========================================="
echo ""

# 1. 编译
echo "[1/5] 编译SITL..."
./waf configure --board sitl > /dev/null 2>&1
./waf copter

if [ ! -f "build/sitl/bin/arducopter" ]; then
    echo "❌ 编译失败"
    exit 1
fi
echo "✅ 编译成功: $(ls -lh build/sitl/bin/arducopter | awk '{print $5}')"
echo ""

# 2. 启动SITL
echo "[2/5] 启动SITL (后台运行)..."
Tools/autotest/sim_vehicle.py -v ArduCopter -f wuk --console \
    --add-param-file=Tools/autotest/default_params/copter-wuk.parm \
    > sitl_output.log 2>&1 &
SITL_PID=$!
echo "✅ SITL PID: $SITL_PID"
sleep 15
echo ""

# 3. 执行自动化测试
echo "[3/5] 运行自动化测试..."
python3 Tools/autotest/sitl_wuk_test.py
echo ""

# 4. 检查日志
echo "[4/5] 检查日志文件..."
if [ -f "wuk_gb.log" ] && [ -f "wuk_gb.jsonl" ]; then
    echo "✅ 日志文件生成成功"
    echo "  - wuk_gb.log: $(wc -l < wuk_gb.log) 行"
    echo "  - wuk_gb.jsonl: $(wc -l < wuk_gb.jsonl) 行"
else
    echo "❌ 日志文件未生成"
fi
echo ""

# 5. 停止SITL
echo "[5/5] 停止SITL..."
kill $SITL_PID 2>/dev/null || true
echo "✅ 测试完成"
echo ""

echo "========================================="
echo "查看日志: cat sitl_output.log"
echo "查看测试数据: cat wuk_gb.log"
echo "========================================="
```

### 11.2 运行
```bash
chmod +x full_test.sh
./full_test.sh
```

---

## 十二、关键指标

### 性能指标
- **编译时间**: ~3-5秒 (增量编译)
- **SITL启动时间**: ~10-15秒
- **变形总时长**: ~5-6秒
  - Phase 1 (0-45°): ~3秒 (15°/s)
  - Phase 2 (停留): 2秒
  - Phase 3 (45-90°): ~3秒 (15°/s)

### 数据输出
- **SITL打印频率**: 每100ms (10Hz)
- **日志记录频率**: 每200帧
- **JSON日志大小**: 约500KB/次完整测试

### 通信延迟
- **RC输入 → UART消息**: <10ms
- **FC角度更新 → SITL读取**: <1ms (全局变量)
- **PWM计算 → 电机响应**: <1帧 (~1ms)

---

## 附录A: 常用命令速查

```bash
# 编译
./waf configure --board sitl && ./waf copter

# 启动SITL
./test_wuk_sitl.sh

# MAVProxy基本命令
mode LOITER          # 切换模式
arm throttle         # 解锁
rc 3 1600           # 油门起飞
rc 7 1900           # 测试RC7
rc 9 1900           # 触发变形
status              # 查看状态
disarm              # 上锁
exit                # 退出

# 日志查看
cat wuk_gb.log | less
python3 -m json.tool < wuk_gb.jsonl | less
grep "angle=45" wuk_gb.log

# 数据分析
./tools/wuk_log_parser wuk_gb.log
python3 tools/wuk_plot.py
```

---

## 附录B: 预期时间轴

```
T=0s     : rc 9 1900 触发
T=0.1s   : 模式切换到MORPH
T=0.2s   : FC angle=0°, SITL开始响应
T=0.3s   : FC angle=1.5°
T=1.0s   : FC angle=15°
T=2.0s   : FC angle=30°
T=3.0s   : FC angle=45° (到达)
T=3.0-5.0s : 停留在45° (2秒)
T=5.1s   : FC angle=46.5° (开始继续)
T=6.0s   : FC angle=60°
T=7.0s   : FC angle=75°
T=8.0s   : FC angle=90° (到达)
T=8.1s   : 模式切换到GROUND
```

---

**文档版本**: v1.0  
**最后更新**: 2025年12月10日  
**适用版本**: ArduPilot 4.x with WuK modifications
