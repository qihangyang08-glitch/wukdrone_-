

### 第一步：安装 VS Code (在 Windows 上)

如果你电脑上还没有 **Visual Studio Code**：
1.  去官网 [code.visualstudio.com](https://code.visualstudio.com/) 下载 Windows 版并安装。
2.  安装时一路“下一步”即可。

### 第二步：安装 WSL 插件 (关键！)

1.  打开你刚装好的 **VS Code**。
2.  点击左侧边栏的 **方块图标 (Extensions/扩展)**（或者按 `Ctrl+Shift+X`）。
3.  搜索框输入：`WSL`。
4.  找到微软官方出的那个 **WSL** 插件（蓝色图标），点击 **Install (安装)**。

### 第三步：用 VS Code 打开 Ubuntu 里的代码

这一步最帅。请回到你的 **Ubuntu 黑终端窗口**（PowerShell/Windows Terminal）。

1.  确保你现在的路径是在 ardupilot 文件夹里。输入：
    ```bash
    cd ~/ardupilot
    ```
2.  输入这行神奇的命令：
    ```bash
    code .
    ```
    *(注意：code 后面有一个空格，然后是一个点。意思是“用 VS Code 打开当前目录”)*

3.  **见证奇迹：**
    *   Windows 会自动弹出一个 VS Code 窗口。
    *   左下角会显示绿色的 `WSL: Ubuntu`。
    *   左侧的文件列表里，直接显示出了 Linux 里的所有代码文件！

---

### 第四步：在 VS Code 里精准修改代码

现在你有了现代化的编辑器，找 `setup` 函数易如反掌。

1.  **打开文件：**
    在左侧文件树里，点开 `ArduCopter` 文件夹，找到 `Copter.cpp`，点击打开。

2.  **搜索函数：**
    *   点击代码编辑区域，按 `Ctrl + F`。
    *   输入：`void Copter::setup`。
    *   它会高亮显示这个位置（大概在第 300 行左右）。

3.  **修改代码：**
    在 `void Copter::setup()` 下面的大括号 `{` 后面，插入你的代码：

    ```cpp
    void Copter::setup() 
    {
        // 你的代码插在这里
        hal.console->printf("\n\n--------------------------------\n");
        hal.console->printf("WARNING: WuK System is Online!!!!\n");
        hal.console->printf("--------------------------------\n\n");
        
        // 下面是原有的代码，不要动
        // Load the recently booted parameter
        // ...
    ```

4.  **保存：** 按 `Ctrl + S`。

---

### 第五步：在 VS Code 里编译与运行

你不需要再切回那个黑窗口了，VS Code 里也能开终端。

1.  **打开内嵌终端：**
    在 VS Code 顶部菜单栏，点击 `Terminal (终端)` -> `New Terminal (新建终端)`。
    *(屏幕下方会弹出一个小黑框，这里就是 Ubuntu 的命令行)*

2.  **编译：**
    输入：
    ```bash
    ./waf copter
    ```

3.  **运行仿真：**
    编译成功后，输入：
    ```bash
    sim_vehicle.py -v ArduCopter
    ```

