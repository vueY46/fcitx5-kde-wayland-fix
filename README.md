# KDE Plasma 6 Wayland 下修复 Electron 应用（Chromium / VS Code）无法使用 fcitx5 输入中文

## 问题表现

- Konsole、Firefox 等原生应用可以正常输入中文
- Chromium、VS Code、Clash Verge 等 Electron / Chromium 系应用完全无法切换输入法

## 根本原因

KDE Plasma Wayland 下，普通应用通过 D-Bus 前端连接 fcitx5，而 Electron 系应用使用 **Wayland text-input-v3 协议**。这个协议需要 KWin 作为中间人转发给输入法，但 KWin 必须正确配置才知道把输入请求交给谁。

同时，全局设置 `GTK_IM_MODULE` 和 `QT_IM_MODULE` 会强制应用走老的 IM 模块路径，与 KWin 的 text-input 协议冲突。

## 修复步骤

### 1. 设置 KWin 使用 fcitx5 作为输入法后端

```bash
kwriteconfig6 --file kwinrc --group Wayland --key InputMethod \
  /usr/share/applications/fcitx5-wayland-launcher.desktop
```

**注意**：必须是 `fcitx5-wayland-launcher.desktop`，不是 `org.fcitx.Fcitx5.desktop`。前者会让 fcitx5 以正确模式对接 Wayland text-input-v3 协议。

### 2. 删除多余的全局 IM 环境变量

检查这些文件并删除以下行：

- `~/.bashrc`
- `~/.zshrc`
- `~/.xprofile`
- `/etc/environment`
- `~/.config/environment.d/*.conf`

需要删除的内容：

```bash
GTK_IM_MODULE=fcitx    # 删除！
QT_IM_MODULE=fcitx     # 删除！
```

**只保留** `XMODIFIERS`，放在 `~/.config/environment.d/fcitx5.conf`：

```ini
XMODIFIERS=@im=fcitx
```

或者用 Plasma 环境脚本 `~/.config/plasma-workspace/env/fcitx5.sh`：

```sh
#!/bin/sh
export XMODIFIERS="@im=fcitx"
```

### 3. 生效

注销并重新登录，或者至少重启 KWin：

```bash
kwin_wayland --replace &
```

## 可选：Chromium 启动参数

如果还有问题，在 `~/.config/chromium-flags.conf` 中加入：

```
--enable-features=UseOzonePlatform
--ozone-platform=wayland
--enable-wayland-ime
--wayland-text-input-version=3
```

VS Code 在 `~/.config/code-flags.conf` 中加入：

```
--ozone-platform-hint=auto
--enable-wayland-ime
--wayland-text-input-version=3
```

## 验证

```bash
fcitx5-diagnose 2>/dev/null | grep -E "IC \[.*chromium|IC \[.*code"
```

能看到 Chromium 或 code 的 InputContext 即表示成功。

## 相关链接

- [Arch Wiki - Fcitx5](https://wiki.archlinux.org/title/Fcitx5)
- [fcitx5-wayland-launcher](https://github.com/fcitx/fcitx5/wiki/Using-Fcitx-5-on-Wayland)

---

**环境**：Arch Linux + KDE Plasma 6 + Wayland + fcitx5 + Rime
