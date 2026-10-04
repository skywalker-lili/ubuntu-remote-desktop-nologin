# Ubuntu 远程桌面免物理登录配置（GNOME）

在**无人登录、显示器/电视关闭**的状态下，用手机 / Windows App 通过 RDP 直接远程接入 Ubuntu GNOME（Wayland）桌面的实战配置，附完整踩坑记录。

> **文档更新日期：2026-10-04。** 文中所有 bug 的状态均以这一天为准。如果你看到本文时，GNOME「Remote Login」的会话交接 bug 已经修复，请重新评估第四节的结论并更新修订记录。
>
> English version: [README.md](README.md)
>
> 环境快照：**Ubuntu 26.04 LTS / GNOME Shell 50.1（Wayland）/ gnome-remote-desktop 50.0**，AMD Ryzen 7 7735HS 迷你主机。下述架构可迁移到任何较新的 GNOME + gnome-remote-desktop 组合。

## 一、目标

外出时打开手机，直接连回家里的迷你主机——**机器无人登录、屏幕关着**也能接入并看到桌面。

## 二、五层架构

每层只解决一个特定的失效模式，缺任何一层都会退回「必须有人在机器前」的旧世界：

### 第 1 层：HDMI dummy plug（保输出）
HDMI 口插一个假负载（4K@60 EDID 模拟器），让操作系统始终认为有显示器在线——电视关机或切走输入源时，合成器依然保有逻辑显示器。电视 + 假负载镜像模式下，会话一动电视就亮。

### 第 2 层：GDM 自动登录（保会话）
`/etc/gdm3/custom.conf`：
```ini
[daemon]
AutomaticLoginEnable=true
AutomaticLogin=youruser
```
保证开机后用户会话必然存在。Desktop Sharing 守护进程依附于用户会话：没有会话，就没有服务。

### 第 3 层：开机即锁（补安全）
自启动项（`~/.config/autostart/lock-on-login.desktop`）在登录后执行 `sleep 5 && loginctl lock-session`。自动登录的代价是「开机即敞开桌面」，用延时锁屏补回来：会话先建立（服务拉起），5 秒后锁门。

### 第 4 层：allow-locked-remote-desktop 扩展（放行）
GNOME 扩展 [allow-locked-remote-desktop](https://github.com/jikamens/allow-locked-remote-desktop) 给 gnome-remote-desktop 的 `session-modes` 加入 `unlock-dialog`。锁屏状态下 RDP 连入不再被踢，而是看到解锁界面。

注意：GDM 登录页与解锁界面在照片里**一模一样**，判断依据是 `loginctl list-sessions`，不是看图。

### 第 5 层：Desktop Sharing 3390 + 钥匙串空密码（服务）
- 用户级 Desktop Sharing 监听 **3390**（若同时开启 Remote Login，系统级会占用 3389，用户级自动挪到 3390）。在你的 VPN/ACL 里放行该端口（本文环境是 Tailscale：`tcp:3390`）。
- **GNOME login keyring 设为空密码**（seahorse「密码和密钥」→ Login 钥匙串 → 更改密码 → 留空）。自动登录没有密码可输，带密码的钥匙串会一直锁着，gnome-remote-desktop 读不到存储的凭据，客户端收到 `Credentials are not set, denying client`。空密码钥匙串在任何登录方式下都自动解锁。

### 客户端
- Windows App 没有端口输入框：「电脑名称」直接写 `IP:端口`（如 `100.x.y.z:3390`）。
- 备选客户端 **aRDP**（开源，FreeRDP 系）：连接失败会快速报错 + 一键重试，不会挂在死连接上无限转圈（见坑 3）。

## 三、为什么不用 Remote Login（3389）

GNOME 有两种远程模式：**Desktop Sharing**（接管当前用户桌面）和 **Remote Login**（从 GDM 登录页登进系统）。后者听起来才是「无人值守」的正解，**但截至 2026-10-04 它有致命 bug，不要依赖它**：

1. **会话交接黑屏 bug**：若系统里已存在你的用户会话，从 Remote Login 的 GDM 登录页登录时会提示 "session already running / force stop"。选择强制接管后，大概率进入**全黑屏幕、只有鼠标可拖动**的状态。日志显示 GDM 把 RDP 客户端交接给用户会话时断裂，还会泄漏多个 `gdm-greeter` 僵尸会话。
2. **凭据验证不稳定**：同一套用户名密码，Desktop Sharing 能进，Remote Login 却反复报 `Could not find user in SAM database` / `MIC verification failed`。
3. **登录页点击失灵**：通过 RDP 连到 GDM 登录页时，点击账号偶尔无反应，需要断开重连「碰运气」。

**结论（2026-10-04）**：GNOME Remote Login 与「已存在的用户会话」配合不可靠。五层架构刻意绕开它——自动登录保证会话永远存在，Desktop Sharing 直接接管，全程不经过 GDM 交接那条坏路。

> 若日后 GDM 交接 bug 修复，Remote Login 可作为「无人值守 + 真登录页」的更优解回归（可以不开自动登录、钥匙串恢复带密码）。在那之后读到本文的朋友，请更新修订记录。

## 四、踩坑记录（症状 → 根因 → 修复）

### 坑 1：Windows App 报 0x104「找不到远程电脑」
- **不是网络问题**：是没有人登录 → 用户会话不存在 → Desktop Sharing 守护进程不存在。而自动登录本身是死的：GDM 日志 `Session never registered, failing`，会话启动约 1 秒即亡。
- **真凶**：某个开机自启的用户级 systemd 单元写了 `Wants=graphical-session.target` + `WantedBy=default.target`——它一开机就把 target 顶起来，真正的 gnome-session 启动时看到 target 已激活，误判「已有图形会话」当场退出。这是竞态，时好时坏。
- **修复**：会话级单元必须由 target 拉起（`WantedBy=graphical-session.target` + `PartOf=`），绝不能反过来自己拉 target（`Wants=`）。

### 坑 2：锁屏连入报「凭据未设置」/ 一直拒绝客户端
- **根因**：自动登录不输密码 → `gkr-pam: couldn't unlock the login keyring` → 钥匙串锁死 → RDP 服务端读不到存储的凭据（`[RDP] Credentials are not set, denying client`）。手动登录的机器永远碰不到这个问题，极具迷惑性。
- **修复**：login keyring 改空密码（第 5 层）。
- **验证方法**：用 Secret Service 协议验证，别信 `gnome-keyring-daemon --unlock`（它不管密码对错都返回 exit 0）。正确姿势：对 `org.freedesktop.Secret.Service` 调 `Lock` 再调 `Unlock`（参数是 `ao` 数组，集合路径 `/org/freedesktop/secrets/collection/login`）。若 `Unlock` 返回 `prompt="/"` 且集合进入解锁列表，即证明空密码生效。

### 坑 3：Windows App 卡在「正在保护到远程电脑的连接…」永远不动
- **根因**：服务端日志 `BIO_read retries exceeded` → `ERRCONNECT_CONNECT_TRANSPORT_FAILED`——TCP 拔通了但客户端一个字节都没发过来，传输层已死，页面上等多久都不会活。多发于手机射频刚唤醒 / VPN 路径切换（中继 ↔ 直连）的瞬间：握手首包掉进缝里。
- **修复**：服务端无从修。**卡住立即掐掉重试**——日志证明失败后 1 秒的重试直接成功。等待的成本是 30 秒超时，重试的成本是 1 秒。aRDP 会快速失败 + 一键重试，体验更好。

### 坑 4：Windows App 设置里找不到端口
「电脑名称」直接写 `IP:端口` 即可，没有单独的端口字段。

### 坑 5：锁屏后电视「搜寻信号」/ 黑屏
正常现象：锁屏屏保会给所有输出 DPMS 断电（dummy plug 一起黑）。RDP 走合成器内部截屏管线，不依赖物理输出供电；远程连入会看到解锁界面，解锁后电视随之点亮。

## 五、验收清单

1. 重启后不碰任何东西 → 自动落桌面，5 秒后锁屏（电视黑屏正常）；
2. 手机连 `IP:3390` → 出解锁界面 → 输密码 → 桌面；
3. 解锁后电视镜像点亮，会话级服务自动就绪；
4. 锁屏状态下重复第 2 步仍可连入（扩展放行）。

## 六、修订记录

- **2026-10-04**：初版。Ubuntu 26.04 LTS / GNOME Shell 50.1 / gnome-remote-desktop 50.0。Remote Login 交接 bug 状态：**未修复**。
