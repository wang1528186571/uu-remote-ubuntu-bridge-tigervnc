# UU Remote Ubuntu Bridge — TigerVNC Edition

> **Fork 说明**：本仓库是 [lachlanchen/uu-remote-ubuntu-bridge](https://github.com/lachlanchen/uu-remote-ubuntu-bridge) 的社区适配版（MIT）。
> 上游项目的完整架构、工作原理、安全设计与文档全部继承自上游，请先阅读[上游 README](https://github.com/lachlanchen/uu-remote-ubuntu-bridge#readme) 与[中文说明](i18n/README.zh-Hans.md)。
> 本 fork 只解决一个问题：**上游的 VNC 中继路径所依赖的经典 RealVNC Viewer 已经停售，全新安装无法走通**。我们把 viewer 换成开源的 TigerVNC，并修复了随之而来的全部兼容问题。

---

## 为什么需要这个 fork（上游现在装不上）

上游 `--desktop-relay vnc` 路径在设计时使用**经典版 RealVNC Viewer（6.x/7.x）**，脚本硬编码了它的专有启动参数与窗口标题。2026 年 RealVNC 调整产品线后，这条路径对全新安装彻底失效：

1. **经典版 VNC Viewer 已停发**。realvnc.com 下载页全面换成新的 RealVNC Connect Viewer 8.x，经典版 7.x 的 deb 在官方 CDN 已无法下载（含 Web Archive 也无有效存档）。
2. **RealVNC Connect Viewer 8.x 不兼容上游脚本**：
   - 它拒绝上游传的全部经典参数（`-AllowMainClose`、`-FullScreenHint`、`-EnableToolbar`、`-GrabKeyboard`、`-ShowSplash` 等），直接报 `Bad argument` 退出；
   - 窗口标题从 `<host>:<port> - RealVNC Viewer` 变成了固定的 `RealVNC Connect Viewer`，上游的窗口匹配正则失效；
   - 最致命的是它把**直连第三方 VNC 服务器也挡在 RealVNC 云账号登录墙后**（未登录时即使命令行给出地址也不发起 TCP 连接），而本桥接的 viewer 跑在无人交互的虚拟显示器上，根本无法完成登录。
3. 结果：全新机器执行上游 `./install.sh --desktop-relay vnc` 后服务崩溃循环，报 `The VNC viewer did not create the Ubuntu desktop relay window`。

## 本 fork 的改动（共 5 处，见 `git log`）

| # | 文件 | 改动 | 原因 |
|---|------|------|------|
| 1 | `scripts/uu-remote-bridge` (~L1455) | viewer 改为 `/usr/bin/xtigervncviewer -FullScreen -Shared -RemoteResize=0` | TigerVNC 是 Ubuntu 官方仓库自带的开源 VNC 客户端，支持裸 `-FullScreen`、`host:port` 直连，无任何账号要求 |
| 2/3 | `scripts/uu-remote-bridge` (~L1468/L1566) | 两处窗口标题正则改为 `' - TigerVNC$'` | TigerVNC 的窗口标题格式为 `<桌面名> - TigerVNC` |
| 4 | `scripts/verify.sh` (~L184) | viewer 进程 pgrep 模式加入 `xtigervncviewer` | 让上游自验收脚本能识别新 viewer |
| 5 | `scripts/uu-remote-bridge` (~L824) | x11vnc 参数追加 `-cursor arrow -nocursorshape` | TigerVNC 把远端光标画成本地 X 指针，不进入画面帧，导致 UU 抓屏流里**永远看不到鼠标**；禁用光标伪编码后 x11vnc 会把箭头直接合成进像素，任何抓屏都可见 |

其余全部组件（Wine 补丁、input broker、终端桥、systemd 单元、哈希校验、安装器）与上游保持一致。

## 验收

在真实 X11 工作站（Ubuntu 24.04.4 + GNOME 46，`--desktop-relay vnc`）上执行上游自验收：

```
./install.sh --desktop-relay vnc --skip-packages
# => 21 PASS / 0 FAIL (INSTALL_EXIT=0)
```

手机端实测：UDP P2P 连接、完整画面、鼠标键盘双向控制正常。

## 部署必读：三个关键配置（上游文档未强调，实测踩坑）

装完后如果出现"连得上、看得到，但无法操作/显示不全/没有鼠标"，逐条对照：

**1. VNC 中继必须显式指定直连输入路由**（写入 `~/.config/uu-remote-bridge/environment`）：

```
UURB_KEYBOARD_ROUTE=x11
```

默认的 `rdp` 路由在 VNC 中继模式下没有注入后端，症状是画面完全正常但鼠标键盘全部无效。诊断：`tail` Wine 前缀下 `AppData/Local/Temp/uu-input-bridge.log`，失败行为 `route=broker result=0 error=21`，成功为 `result=1 error=0`。

**2. 画布分辨率必须匹配桌面分辨率**：

```
UURB_RESOLUTION=2560x1440          # 与被控桌面一致
UURB_FOLLOW_DESKTOP_RESOLUTION=on  # 或开启自动跟随
```

画布小于桌面时画面被缩放裁剪，且输入按 1:1 坐标注入导致点击位置全部偏移。

**3. 被控机必须关闭 GNOME 自动锁屏**：

```
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.desktop.session idle-delay 0
```

锁屏会吃掉全部注入输入并把画面变黑——症状与"输入失效"一模一样，极易误诊。

## 安装

```bash
git clone https://github.com/<this-repo>.git
cd uu-remote-ubuntu-bridge-tigervnc
./install.sh --desktop-relay vnc
```

安装器行为与上游一致：只从网易官方域名 `uuyc.163.com` 下载客户端、强制校验 SHA-256、拒绝未知二进制。唯一区别是 VNC 中继使用系统 TigerVNC，不再需要 RealVNC。

管理命令：`uu-remote status|restart|console|logs`。

## 兼容性说明

- 被控端：Ubuntu 24.04 + GNOME 46 + **X11 会话**（Wayland 请用上游默认 RDP 路径）
- 已知取舍：X11/VNC 路径无图片/文件剪贴板、无隐私屏（与上游 X11 验收一致）
- 上游的 `--desktop-relay rdp`（Wayland）路径未改动，仍按上游文档执行

## 致谢与许可

- 全部核心工程（Wine 兼容层、输入代理、安全设计、验收体系）由 [lachlanchen/uu-remote-ubuntu-bridge](https://github.com/lachlanchen/uu-remote-ubuntu-bridge) 完成，请给上游点星。
- 本 fork 遵循上游的 [MIT License](LICENSE)（Copyright (c) 2026 Lachlan Chen）。
- UU 远程是网易的产品；本仓库不含也不分发任何网易二进制，安装器仅从官方域名下载并做哈希校验。
