# 🍓 树莓派折腾日志 Day 3
### 显示屏上机、Wi-Fi 联网，硬件平台初步成型

> **背景：** Psychology × CS 跨界学生。前两天解决了系统烧录和 SSH 连接，今天第一次让树莓派真正"独立"起来——接上显示屏，联上网，不再依赖 Mac 作为中转。

| 项目 | 详情 |
|------|------|
| 📅 日期 | 2026年6月25日 |
| 🖥️ 设备 | 树莓派5（4GB）|
| 🎯 今日目标 | 显示屏连接 + Wi-Fi 联网 |
| ✍️ 作者 | Dora（YCUA-765）|

---

## 🖥️ 显示屏：7寸 HDMI 触控屏

### 为什么要买显示屏

Day 2 虽然通过手机热点让 Mac 和树莓派进入同一网段，成功建立了 SSH 连接，但实际使用中发现一个问题：SSH 连接期间，在 Mac 或手机上向 Claude 提问（比如查树莓派相关问题）时，Claude 完全无法响应。

推测原因是手机热点的网络资源被树莓派的 SSH 连接占用，导致同一热点下其他设备的网络请求无法正常发出。（注：这方面网络知识尚不足以确认根因，暂时以此假设为前提。）

实际效果是：Mac 和手机都能连上热点，但实际上网几乎不可用。在无法查资料、无法问问题的情况下，SSH 远程连接的开发体验极差。

**解决思路：给树莓派配一块独立显示屏。**

有了显示屏，树莓派就变成一台独立运行的设备，不再需要依赖 Mac 做中转，热点资源可以完全留给 Mac 和手机正常使用。

---

### 硬件规格

```
尺寸：7寸
接口：HDMI IN（视频信号）+ CTOUCH / USB（触控信号 + 供电）
分辨率：1024 × 600
触控：电容触摸，免驱
```

### 接线方式

这块屏幕需要两根线同时连接：

```
树莓派 micro-HDMI 0 → （micro-HDMI 转 HDMI 线）→ 屏幕 HDMI IN 口   ← 视频信号
树莓派 USB-A 口      → （USB 线）                → 屏幕 CTOUCH 口   ← 触控信号 + 供电
```

> ⚠️ 树莓派5有两个 micro-HDMI 口，优先接 **HDMI 0**（靠近 USB-C 电源口的那个），这是主显示输出。

### 踩坑：接口认知错误

连接过程卡了一段时间，根本原因是对这几个接口完全陌生：

```
micro-HDMI   ← 树莓派5的视频输出口（不是标准 HDMI，需要转接）
HDMI IN      ← 屏幕的视频输入口
CTOUCH       ← 屏幕的触控 + 供电 USB 口
Backlight CTR / Power ← 背光控制和供电，不传数据
```

初次看到屏幕背面时，对这几个接口完全没有概念，不知道任何一个的实际用途，也无从猜测哪个是视频输入、哪个是触控、哪个只是供电。查资料和询问之后才搞清楚：`HDMI IN` 是接收树莓派视频输出的口，`CTOUCH` 是触控数据 + 辅助供电的 USB 口，`Backlight CTR` 和 `Power` 只管背光和供电，不传任何数据。接口命名是从**屏幕自身视角**定义的，不是从树莓派视角。

**这算哪个领域的知识欠缺？**

不确定这类接口知识在传统工科教育（比如教科书或课堂）里是否有系统覆盖，还是本来就属于"要接触实物才会知道"的范畴。但有一点感受是确定的：看到实物之后，这些接口的名字和功能就很难忘了。可能有些知识天然适合从实物出发去学，而不是从定义出发。

### 结果

开机后**无需修改任何配置**，屏幕自动识别分辨率，直接显示桌面 ✅  
触控功能同步可用，点击响应正常 ✅

> 说明书要求在 `config.txt` 里手动写入分辨率参数。但实测树莓派5 + Bookworm 系统已能自动协商，无需手动配置。若遇到黑屏或分辨率异常，再按说明书添加对应参数即可。

---

## 📷 摄像头：120° 广角 MIPI-CSI

今天完成了 120° 广角摄像头的物理安装，接入树莓派5的 CSI 接口。

```
状态：已安装，待测试
接口：MIPI-CSI（树莓派5上的排线接口）
```

测试命令备用：

```bash
libcamera-hello    # 验证摄像头是否被识别，启动实时预览
```

---

## 🌐 Wi-Fi 联网：nmcli 踩坑记

有了显示屏之后，通过 SSH 进入终端，尝试连接手机热点。

### 过程记录

**Step 1：直接连接，失败**

```bash
sudo nmcli dev wifi connect "iPhone" password "密码"
# → Error: No network with SSID 'iPhone' found
```

原因：nmcli 在连接前需要先扫描，缓存里没有这个网络。

**Step 2：手动扫描**

```bash
sudo nmcli dev wifi list
# → 列表中出现 iPhone 热点 ✅
```

**Step 3：再次连接，新报错**

```bash
sudo nmcli dev wifi connect "iPhone" password "密码"
# → Error: 802-11-wireless-security.key-mgmt: property is missing
```

尝试手动指定加密方式 `key-mgmt wpa-psk`，报 `invalid extra argument`——nmcli 不接受这种写法。

**Step 4：连接成功但 IP 分配失败**

```bash
sudo nmcli dev wifi connect "iPhone" password "密码"
# → Error: Connection activation failed: IP configuration could not be reserved
```

握手成功，但获取 IP 失败。

**根因：VPN 干扰**

手机开着 VPN，热点的 DHCP 服务无法正常分配 IP 地址给树莓派。关闭 VPN 后重试：

```bash
sudo nmcli dev wifi connect "iPhone" password "密码"
# → Device 'wlan0' successfully activated ✅
```

> 💡 **规律**：手机开 VPN 时开热点，VPN 会接管网络栈，导致热点 DHCP 异常。树莓派能完成 Wi-Fi 握手，但拿不到合法 IP。遇到 `IP configuration could not be reserved`，先检查热点设备的 VPN 状态。

### 关于密码缓存

第一次连接时输入过密码，nmcli 会自动保存这个网络配置。之后再连同一热点，无需重新输入密码，直接执行连接命令即可。

---

## 💭 今日总结

**技术层面**

- **接口命名视角**：屏幕上的 `HDMI IN` 是从屏幕自身视角定义的，对应树莓派的输出
- **nmcli 连接顺序**：`list` 扫描 → `connect` 连接，不能跳过扫描步骤
- **VPN 与热点冲突**：VPN 开启时热点 DHCP 可能失效，报 `IP configuration could not be reserved`

**项目进度**

树莓派5现在是一台真正独立运行的设备：有显示输出、有触控输入、有网络连接。下一步是测试摄像头，进入 Project 3 的实质开发阶段。

---

## 📊 当前状态

| 项目 | 状态 |
|------|------|
| 7寸 HDMI 触控屏 | ✅ 显示正常，触控可用 |
| 120° 广角摄像头 | ⏳ 已安装，待测试 |
| Wi-Fi 联网 | ✅ 手机热点连接成功 |
| 桌面环境 | ✅ 可直接操作，不依赖 SSH |

---

*← [Day 2：三次烧录、Bookworm 新坑与"盲目寻址"的教训](./day2.md)*  
*→ Day 4：摄像头测试与 libcamera 环境配置（待更新）*
