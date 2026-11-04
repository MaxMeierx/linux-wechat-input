# VType Linux 下次安装注意事项（踩坑版）

> 用途：下次安装或重装时先看这份。详细安装步骤仍以仓库根目录的 `AGENTS.md` 为准。
>
> 本次环境：**Arch Linux / KDE Plasma / Wayland / x86_64 / Fcitx5 5.1.23**。

---

## 1. 安装前

### 1.1 先读 `AGENTS.md`

- 不要直接照着 README 跳步骤。
- 先检查架构、发行版、桌面、会话类型、Fcitx5 版本和当前输入法。
- 这是非官方兼容方案，x86_64 通过 QEMU 运行 Android ARM64 引擎，确认用户接受性能开销和升级维护成本。

### 1.2 sudo 不要交给自动化终端处理

本次安装中，自动化终端无法直接输入 sudo 密码，也没有弹出密码提示。

下次注意：

- 需要 sudo 时先停下来，让用户在图形终端输入密码；
- 或者由用户明确同意后使用 `pkexec`；
- 不要在后台反复尝试 sudo，也不要让安装卡住等待密码。

### 1.3 第一次构建会下载很多东西

`prepare_ndk.sh`、AOSP 运行库和官方 APK 都会下载较大的文件，APK 约 214 MB。网络慢时不是卡死，不要反复删除缓存重来。

### 1.4 不要忽略 locale

本次自动化终端的 `LANG=C.UTF-8` 可能导致 Fcitx5 找不到微信输入法，而图形会话的 systemd 用户环境是 `zh_CN.UTF-8`。

安装或诊断时先确认：

```bash
echo "$LANG"
systemctl --user show-environment | grep -E '^(LANG|LC_)'
```

不要因为自动化 shell 里输入法列表异常就判断安装失败。

---

## 2. 编译与安装

### 2.1 依赖一次装齐

Arch 环境需要确保这些工具可用：

- `base-devel`
- `cmake`
- `fcitx5`
- `qemu-user-static`
- `python`
- `openssl`
- `curl`
- `unzip`
- `e2fsprogs`

缺依赖时先说明并询问，不要在安装脚本失败后才发现。

### 2.2 安装完成后先别急着宣告成功

至少检查：

```bash
pgrep -a fcitx5
fcitx5-remote -n
ls -l "$HOME/.local/lib/wetype-ime/arm64"
```

并运行仓库提供的引擎自检。自检通过后再继续配置输入法。

### 2.3 安装 AppImage 时注意 APK 下载

`./WeTypeIME-Engine-x86_64.AppImage install --user` 会从腾讯官方源下载 APK 并校验 SHA256。没有 FUSE 时使用：

```bash
APPIMAGE_EXTRACT_AND_RUN=1 ./WeTypeIME-Engine-x86_64.AppImage install --user
```

不要手工用未校验的 APK 替换安装目录中的引擎。

---

## 3. 启用 Fcitx5 输入法

### 3.1 修改 Fcitx5 配置前必须先问

涉及以下操作时不要直接执行：

- 编辑 `~/.config/fcitx5/profile`
- 编辑 `~/.config/fcitx5/config`
- 执行 `fcitx5-remote -e`
- 重启或重新加载 Fcitx5

这些操作可能打断用户正在输入的内容。

### 3.2 安装新插件后通常需要让 Fcitx5 重新加载

安装完 AppImage 和插件后，如果输入法设置里看不到“微信拼音”，先确认插件路径和配置已经安装，再征得用户同意后重新加载 Fcitx5。

最终确认：

```bash
fcitx5-remote -n
```

应输出：

```text
wetype-im
```

### 3.3 候选词每页 10 个

本次配置写在：

```text
~/.config/fcitx5/config
```

内容：

```ini
[Behavior]
DefaultPageSize=10
```

注意：

- 这是 Fcitx5 的 Linux 端设置，不会修改手机；
- 第 10 个候选使用数字键 `0`；
- 设置后已用 `fcitx5-remote -r` 重新加载；
- 下次不要为此直接重启整个 Fcitx5，先问用户。

---

## 4. 浏览器在 Wayland 下切换不了输入法

### 4.1 实际踩到的现象

在 KDE Wayland 下：

- Chromium 进程使用 `--ozone-platform=wayland`；
- Fcitx5 的输入上下文列表里没有 Chromium；
- 输入法完全无法切换；
- Vivaldi 强制使用 XWayland 后可以切换。

这是 Chromium 系浏览器的前端接入问题，不是微信输入法引擎坏了。

### 4.2 当前采用的配置

给出问题的 Chromium 系浏览器单独增加：

```text
--ozone-platform=x11
```

本次已配置：

| 浏览器 | 配置文件 |
|---|---|
| Vivaldi Snapshot | `~/.config/vivaldi-snapshot.conf` |
| Chromium | `~/.config/chromium-flags.conf` |
| Brave Beta | `~/.config/brave-beta-flags.conf` |
| Microsoft Edge Dev | `~/.config/microsoft-edge-dev-flags.conf` |

### 4.3 不要假设所有浏览器都用同一个文件名

Chromium 系不同发行包、稳定版/Beta/Dev 的启动器读取的 flags 文件名可能不同。

下次配置前先看启动脚本实际读取哪个文件，例如：

```bash
head -80 /usr/bin/chromium
head -80 /usr/bin/brave-beta
head -80 /usr/bin/microsoft-edge-dev
```

文件里只写启动参数，不要写 shell 命令。

### 4.4 改完必须完全退出浏览器

只刷新页面或新开标签不会切换后端。必须：

1. 关闭该浏览器所有窗口；
2. 确认主进程退出；
3. 重新启动。

验证：

```bash
pgrep -a -f 'chromium|vivaldi|brave|microsoft-edge'
fcitx5-diagnose 2>&1 | grep -E 'Group|program:(chromium|vivaldi|brave|microsoft-edge)'
```

浏览器应出现在 `Group [x11::0]`，而不是完全不在 Fcitx5 的上下文列表中。

### 4.5 不要为所有应用全局关闭 Wayland

不需要把整个桌面切回 X11。优先只给无法接入 Fcitx5 的浏览器配置 XWayland。

本次在当前 KDE/Fcitx5 环境中，Chrome/Chromium 系的 `--enable-wayland-ime` 没有解决问题，所以使用 XWayland 作为兼容方案。

### 4.6 不要因为浏览器问题重装 VType

如果 `fcitx5-diagnose` 里完全看不到浏览器，说明应用还没有接入 Fcitx5。此时重装微信引擎无效，应先修浏览器启动参数。

---

## 5. 关联设备、云候选和词库同步

### 5.1 关联码必须由用户自己输入

关联时需要六位关联码，并回到官方微信输入法客户端确认。

- 不要让自动化脚本猜测或代替用户确认；
- 不要把关联码写进文档、日志或仓库；
- 需要输入关联码时先停下来询问用户。

### 5.2 设备身份不能外传

敏感文件：

```text
~/.local/state/wetype-ime/account
```

它相当于登录凭据，不要读取、打印、复制或上传。

个人词库和日志也可能包含私人内容：

```text
~/.local/share/wetype-ime
~/.local/state/wetype-ime/harness.log
```

排错时只摘取必要错误行，并先让用户确认。

### 5.3 云功能网络可能被代理挡住

云候选、词库同步需要直连官方服务器。代理只认系统级 TUN，不读取普通 `http_proxy` / `https_proxy` 环境变量。

网络异常时先检查直连或 TUN，不要先改代理放行规则。

### 5.4 同步不是实时的

词库拉取和上传由引擎在输入时触发，不会点一下立即同步。刚关联后不要马上判断失败，正常输入一段时间再看状态。

---

## 6. 跨设备剪贴板

### 6.1 默认不要开启图片同步

本次只开启了纯文本剪贴板，图片同步保持关闭。

需要图片同步时必须再次征得用户明确同意，不能顺手加 `--images`。

### 6.2 剪贴板是独立服务

检查：

```bash
python3 ~/.local/lib/wetype-ime/arm64/wetype-cloud.py clipboard status
systemctl --user status wetype-clipboard.service
```

注意：

- 剪贴板默认关闭；
- 启用后新复制的纯文本会发送到官方服务器；
- 首次启动不会上传已有的旧剪贴板；
- 不需要重启 Fcitx5；
- 文件不参与同步。

---

## 7. 更新、重装和卸载

### 7.1 不要随意升级官方 APK

项目固定适配了特定 APK 版本。官方 APK 更新后，以下内容可能需要重新适配：

- 二进制补丁偏移；
- `dict_table.h`；
- SessionConfig 默认值；
- 云请求版本号；
- 资源清单。

没有完成适配和自检前，不要替换已安装的引擎库。

### 7.2 重装时保护用户数据

重装或卸载前确认不要误删：

```text
~/.local/state/wetype-ime/account
~/.local/share/wetype-ime
```

如果确实要清理关联身份或词库，先单独向用户确认。

---

## 8. 下次安装的顺序清单

- [ ] 读仓库根目录 `AGENTS.md`
- [ ] 检查 x86_64、Arch/KDE/Wayland、Fcitx5 版本
- [ ] 确认 sudo 由用户输入
- [ ] 安装依赖
- [ ] 下载 NDK、AOSP 运行库和官方 APK
- [ ] 编译并安装 AppImage
- [ ] 运行引擎自检
- [ ] 征得同意后重新加载 Fcitx5
- [ ] 添加并切换到 `wetype-im`
- [ ] 确认候选窗和选词正常
- [ ] 设置 `DefaultPageSize=10`，第 10 个用 `0`
- [ ] 关联设备时停下来让用户输入关联码
- [ ] 检查云候选和词库同步
- [ ] 按需启用纯文本剪贴板，图片默认关闭
- [ ] 检查 Chromium 系浏览器的 Fcitx5 输入上下文
- [ ] 只给需要兼容的浏览器配置 `--ozone-platform=x11`
- [ ] 完全重启对应浏览器并验证
- [ ] 不读取或传播账号、词库和输入日志

---

## 9. 最短结论

这次最容易踩的坑是：

1. **自动化终端没有 sudo 密码提示**；
2. **Fcitx5 配置不能未经同意修改或重启**；
3. **Chromium 系浏览器原生 Wayland 没有接入 Fcitx5，需要按应用配置 XWayland**；
4. **浏览器 flags 文件名因安装包而异，改完必须完全重启**；
5. **关联码、设备身份、个人词库和输入日志都不能外传**；
6. **图片剪贴板不属于默认配置，必须单独征得同意**。
