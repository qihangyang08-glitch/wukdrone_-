

# 🛠️ 技术笔记：VS Code + WSL2 开发环境网络配置指南

**日期：** 2025年11月28日
**环境：** Windows 11 / WSL2 (Ubuntu) / VS Code / Clash
**目标：** 实现 Host(Windows) 与 Guest(WSL) 的网络互通，解决 Git 下载慢、AI 插件（Gemini/Copilot）无法登录的问题。

### 一、 核心逻辑 (Topology)
*   **WSL2 的特性：** 它是一台独立的虚拟机，有自己的 IP。在 WSL 里访问 `127.0.0.1` 是访问虚拟机自己，而不是 Windows。
*   **解决方案：** 所有网络请求（无论是 Windows 上的 UI 还是 WSL 里的插件），统一指向 **Windows 在 WSL 虚拟网络中的网关 IP**（通常以 `172.` 开头），通过 Windows 上的代理软件出网。

---

### 二、 代理软件设置 (Clash 端)
这是流量的出口，必须配置正确。

1.  **端口 (Port)：** 记下 HTTP/HTTPS 代理端口（通常为 **7890**）。
2.  **允许局域网连接 (Allow LAN)：** **必须开启 (ON)**。
    *   *原因：* 允许来自 WSL 虚拟网卡的外部请求连接代理。
3.  **防火墙 (Firewall)：** 确保 Windows 防火墙允许代理软件通过（调试不通时可暂时关闭防火墙验证）。

---

### 三、 VS Code 设置 (IDE 端)
解决 Gemini Code Assist、Copilot、CodeGeeX 等 AI 插件的登录与联网问题。

**策略：** 采用“大一统”方案，用户区（User）和远程区（Remote）统一使用宿主 IP。

1.  **获取宿主 IP：**
    在 WSL 终端输入：
    ```bash
    ip route show | grep default | awk '{print $3}'
    # 记下这个 IP，例如 172.25.160.1
    ```

2.  **修改设置 (`Ctrl + ,`)：**
    *   搜索 `proxy`，修改以下两处（**填入 `http://<宿主IP>:7890`**）：
        *   `User` (用户) -> `Http: Proxy` : `http://172.xx.xx.1:7890`
        *   `Remote [WSL: Ubuntu]` (远程) -> `Http: Proxy` : `http://172.xx.xx.1:7890`
    *   **Http: Proxy Support** : 选择 `override` (强制覆盖)。
    *   **Http: Proxy Strict SSL** : **取消勾选 (Off)**。
        *   *原因：* 防止 Google/GitHub 因代理证书问题拒绝连接。

3.  **重启生效：** 配置完成后，必须执行 `Reload Window` 或彻底重启 VS Code。

---

### 四、 WSL 终端设置 (命令行端)
解决 `git clone`、`apt install`、`pip install` 的联网问题。

**策略：** 将动态获取 IP 的脚本写入启动文件，实现自动化。

1.  **编辑配置文件：**
    ```bash
    code ~/.bashrc
    ```

2.  **在文件末尾添加以下脚本：**
    ```bash
    # --- WSL2 Proxy Auto-Config ---
    # 1. 动态获取宿主 Windows 的 IP 地址
    export hostip=$(ip route show | grep default | awk '{print $3}')

    # 2. 设置终端代理环境变量
    export http_proxy="http://$hostip:7890"
    export https_proxy="http://$hostip:7890"

    # 3. 设置 Git 全局代理 (可选，防止 git 不走环境变量)
    git config --global http.proxy http://$hostip:7890
    git config --global https.proxy http://$hostip:7890

    # 4. (可选) 打印提示，确认 IP 正确
    # echo "Proxy set to Windows Host: $hostip:7890"
    ```

3.  **生效配置：**
    保存并关闭文件，然后终端输入 `source ~/.bashrc` 或重启终端。

---

### 五、 故障排查命令 (Troubleshooting)

如果某天突然连不上了，按顺序执行以下检查：

1.  **检查连通性 (在 WSL 终端)：**
    ```bash
    curl -v http://<宿主IP>:7890
    ```
    *   *正常：* 返回 Empty reply 或 HTTP 响应。
    *   *异常：* `Connection refused` (防火墙/Allow LAN没开) 或 `Timeout` (IP变了)。

2.  **检查 Git 配置：**
    ```bash
    git config --global --get http.proxy
    ```

---

记录好这份笔记，接下来我们可以心无旁骛地回到 **ArduPilot 仿真代码 (`SIM_Frame.cpp`)** 的开发中了！准备好继续了吗？
