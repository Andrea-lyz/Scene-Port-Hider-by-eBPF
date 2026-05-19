# Scene Port Hider by eBPF

这是一个 KernelSU 模块，用来隐藏 Scene 常用 TCP 端口 `8788`和`8765` 的端口探测。

模块使用 eBPF 在内核侧做端口行为隐藏，当前覆盖：

- `connect()` 型探测：使用 `cgroup/connect4`和`cgroup/connect6` 将非白名单应用对目标端口的本机连接重定向到无服务端口。
- `bind()` 型探测：使用 `cgroup/bind4`和`cgroup/bind6` 将非白名单应用对目标端口的本机绑定临时改为随机端口。
- `bind + getsockname()` 一致性探测：用 kprobe/kretprobe 按 `tgid + fd` 记录原始端口，多次 `getsockname(fd)` 都回填一致结果，并在 `close(fd)` 时清理状态。

模块不使用 `iptables` / `ip6tables`，也不再依赖 `service.d` 脚本。

默认目标应用包名是 Scene：

```sh
com.omarea.vtools
```

## 更新内容

### v2.0.1

- 修复隐藏端口匹配过宽的问题。现在只按 socket API 使用的网络字节序写入目标端口，避免 `8788` 误伤字节翻转后的 `21538`、`8765` 误伤 `15650`。
- 降低无关应用本地服务被误判为检测器的概率。

### v2.0

- 改为纯 eBPF/cgroup socket hook 方案，不再写入 `iptables` / `ip6tables` 规则，也不再依赖 `service.d` 脚本。
- 新增 `cgroup/connect4`、`cgroup/connect6` 处理连接型探测，将非白名单应用对隐藏端口的本机连接重定向到无服务端口。
- 新增 `cgroup/bind4`、`cgroup/bind6` 处理 bind 型探测，将非白名单应用对隐藏端口的本机绑定临时改为随机端口。
- 新增 `bind + getsockname()` 一致性处理，避免检测器通过 `bind()` 后读取实际端口发现改写痕迹。
- 新增 `close()` 清理逻辑，按 `tgid + fd` 清除临时记录，降低长时间运行后的状态残留风险。
- 优化 Scene UID 白名单解析，等待包管理器或运行进程提供真实 UID，避免开机早期误判 UID 导致 Scene 自身无法连接 daemon。
- 新增安装时 BTF 指纹校验，模块包内 `btf/vmlinux.btf` 必须和当前设备 `/sys/kernel/btf/vmlinux` 一致，降低刷错设备包的风险。

## Root 方案兼容性

模块采用通用 Magisk 模块结构，理论兼容：

- KernelSU
- Magisk
- APatch / APM

当前主要在 KernelSU 环境测试通过。

Magisk 和 APatch 用户需要自行确认设备内核满足 eBPF、BTF、cgroup socket hook、kprobe/kretprobe 等要求。这个模块能否正常工作，主要取决于内核能力，而不是 root 管理器本身。

## 重要说明

这个模块和手机内核强相关。不同手机、不同系统版本、不同内核构建出来的模块不一定通用。

推荐每个用户都用自己手机的 `/sys/kernel/btf/vmlinux` 自助构建一次。

不要直接拿别人设备构建出来的包乱刷。

新版本模块会在安装时检查内核 BTF 指纹。如果模块包里的指纹和当前手机 `/sys/kernel/btf/vmlinux` 不一致，安装会被拒绝，避免刷错设备。

## 设备和内核要求

这个模块不是所有 root 设备都能用。建议满足以下条件再尝试：

- 设备是 arm64 / arm64-v8a。
- 已安装 KernelSU，并且 ADB 可以获取 root 授权。
- 当前内核支持 eBPF、BPF map 和 cgroup socket hook。
- 当前内核存在 `/sys/kernel/btf/vmlinux`。
- `/sys/fs/cgroup` 可用，并允许 root 使用 legacy `BPF_PROG_ATTACH` 挂载 `connect4/connect6` 和 `bind4/bind6` 程序。
- 当前内核允许挂载 `bind`、`getsockname`、`close` 相关 kprobe/kretprobe。

普通用户可以先执行：

```sh
su
ls -lh /sys/kernel/btf/vmlinux
exit
```

如果提示文件不存在，当前公开自助构建方案基本不支持这台设备。

如果想进一步检查：

```sh
su
uname -a
getprop ro.product.cpu.abi
mount | grep " /sys/fs/cgroup "
cat /sys/fs/cgroup/cgroup.controllers 2>/dev/null
cat /proc/kallsyms | grep -E "(__sys_bind|__arm64_sys_bind|__sys_getsockname|__arm64_sys_getsockname|__arm64_sys_close)" | head
exit
```

一般建议 Android 12 以后、内核 5.4 以后、有 `/sys/kernel/btf/vmlinux` 的 arm64 设备再尝试。这个版本线不是绝对要求，因为有些厂商会回移植 eBPF/BTF，也有些新内核会裁剪相关能力。

## 普通用户自助构建

这条路线最简单：不需要自己安装 Android Studio、NDK、bpftool 或 libbpf，只需要 Fork 仓库，然后让 GitHub Actions 自动构建。

### 1. Fork 仓库

打开本仓库，点右上角 `Fork`，创建到自己的 GitHub 账号下面。

进入自己 Fork 后的仓库，点 `Actions`。

如果 GitHub 提示启用 Actions，就点启用。

### 2. 从自己的手机导出 BTF

电脑连接手机，确认 ADB 可用：

```powershell
adb devices
```

然后执行：

```powershell
adb shell su -c "cp /sys/kernel/btf/vmlinux /storage/emulated/0/Download/vmlinux.btf && chmod 0644 /storage/emulated/0/Download/vmlinux.btf"
adb pull /storage/emulated/0/Download/vmlinux.btf vmlinux.btf
adb shell su -c "rm -f /storage/emulated/0/Download/vmlinux.btf"
```

注意：不要用下面这种方式导出：

```powershell
adb shell su -c "cat /sys/kernel/btf/vmlinux" > vmlinux.btf
```

这种方式容易把二进制 BTF 文件弄坏。必须使用 `adb pull`。

如果导出时报错，通常是以下原因之一：

- 手机没有 root 权限。
- ADB 没有拿到 root 授权。
- 当前内核没有 `/sys/kernel/btf/vmlinux`。

### 3. 上传 BTF 到自己的 Fork

打开自己 Fork 的 GitHub 仓库。

进入 `btf` 文件夹，点：

```text
Add file -> Upload files
```

上传刚刚导出的：

```text
vmlinux.btf
```

确保上传后的路径是：

```text
btf/vmlinux.btf
```

然后点 `Commit changes`。

### 4. 运行 GitHub Actions 构建

进入自己 Fork 的仓库：

```text
Actions -> Build KernelSU module -> Run workflow
```

第一次建议这样选：

```text
Create a GitHub Release after building: false
```

然后点绿色的 `Run workflow`。

第一次构建会比较久，因为 GitHub Actions 需要下载 Android NDK，并编译 `libbpf`、`libelf` 和 `zlib`。

后续构建会使用缓存，通常会快一些。

### 5. 下载构建好的模块

Actions 成功后，进入那次运行记录。

在页面底部 `Artifacts` 里下载：

```text
ScenePortHider_Release
```

下载后解压，里面会有：

```text
hideSceneport_module.zip
```

这个就是 KernelSU 模块包。

如果运行 workflow 时把 `Create a GitHub Release after building` 设为 `true`，也可以直接在仓库 `Releases` 页面下载 `hideSceneport_module.zip`。

### 6. 安装模块

把 `hideSceneport_module.zip` 放到手机。

打开 KernelSU Manager：

```text
模块 -> 从本地安装 -> 选择 hideSceneport_module.zip -> 重启
```

## 验证是否成功

重启后执行：

```sh
su
cat /data/adb/modules/hideSceneport/hideport.log
ps -A | grep hideport
iptables -S OUTPUT | grep -E "8765|8788"
ip6tables -S OUTPUT | grep -E "8765|8788"
exit
```

正常情况下，`hideport.log` 里应该能看到类似内容：

```text
hidden port: 8788
hidden port: 8765
allowed uid: 0
allowed uid: 1000
allowed uid: 2000
allowed uid: 999
allowed uid: 10384
attached connect4 to /sys/fs/cgroup with legacy single attach
attached connect6 to /sys/fs/cgroup with legacy single attach
attached getsockname probes to __sys_getsockname
attached bind probes to __sys_bind
attached close cleanup probe to __arm64_sys_close
attached bind4 to /sys/fs/cgroup with legacy single attach
attached bind6 to /sys/fs/cgroup with legacy single attach
hideport cgroup-connect loaded
```

`allowed uid` 的具体数字会因设备、用户空间和 Scene 安装方式不同而变化。关键是日志里应同时出现 Scene UI 的真实 UID 和 `scene-daemon` 所需 UID。

同时 `iptables` / `ip6tables` 里不应该再出现本模块写入的 `8765`、`8788` 规则，Scene 应该可以正常打开。

## 修改配置

默认配置在 `hideport.conf`：

```sh
PKG=com.omarea.vtools
PORTS="8788 8765"
ENABLE_EBPF=1
EXTRA_ALLOWED_UIDS=""
WAIT_FOR_UID_TIMEOUT=300
WAIT_FOR_PROCESS=0
```

一般用户不需要改。

配置说明：

- `PKG`：Scene 包名，默认 `com.omarea.vtools`。
- `PORTS`：需要隐藏的本机 TCP 端口。
- `ENABLE_EBPF`：是否启动 eBPF loader。
- `EXTRA_ALLOWED_UIDS`：额外放行 UID。一般留空；如果 Scene 有特殊辅助进程，可以手动填入。
- `WAIT_FOR_UID_TIMEOUT`：等待包管理器或运行进程给出真实包 UID 的秒数。
- `WAIT_FOR_PROCESS`：是否等待 Scene 进程启动后再加载。默认不需要。

模块启动时会等待 `dumpsys package`、`cmd package list packages -U` 或运行进程提供真实 UID。`/data/data/$PKG` 的 owner 只作为额外补充，不会单独触发 loader 启动，避免开机早期只解析到错误 UID 导致 Scene 自己连不上 daemon。

如果 Scene 包名、端口或运行方式发生变化，可以在构建前修改 `hideport.conf`，然后重新构建模块。

## 本地构建方式

如果你不想用 GitHub Actions，也可以在 Linux 或 WSL 中本地构建。

连接手机后执行：

```sh
bash tools/build_for_connected_device.sh
```

脚本会自动：

- 从手机拉取 `/sys/kernel/btf/vmlinux`。
- 生成 `src/vmlinux.h`。
- 下载 Android NDK r25c。
- 编译 Android arm64 的 `libz.a`、`libelf.a` 和 `libbpf.a`。
- 编译 `hideport_loader` 和 `hideport.bpf.o`。
- 打包生成 `../hideSceneport_module.zip`。

## 常见问题

### 为什么别人构建的包我不能直接用？

因为 eBPF CO-RE 依赖目标内核的 BTF 信息。不同手机或不同内核的结构可能不同。

同机型同系统版本有机会通用，但不保证。最稳妥的方式是用自己的手机导出 `vmlinux.btf` 后重新构建。

### Actions 构建出来的文件大小和本地不一样正常吗？

正常。

`hideport_loader` 是静态链接程序，里面包含 `libbpf`、`libelf`、`zlib` 等依赖。不同构建环境生成的二进制大小可能不同。

只要刷入后日志显示 `hideport cgroup-connect loaded`，并且 Scene 正常打开即可。

### 为什么日志里有一些 ENOENT？

模块会按多个候选内核符号依次尝试挂载 kprobe，例如：

```text
__sys_close
__se_sys_close
sys_close
SyS_close
__arm64_sys_close
```

如果某个符号在当前内核不存在，会看到 `-ENOENT`。只要后面出现实际成功的挂载日志，例如：

```text
attached close cleanup probe to __arm64_sys_close
```

就说明功能已经挂上，前面的 `ENOENT` 可以忽略。

### 日志在哪里？

模块日志在：

```text
/data/adb/modules/hideSceneport/hideport.log
```

### 刷入后 Scene 打不开怎么办？

先临时停止 loader：

```sh
su -c 'for p in $(pidof hideport_loader); do kill "$p"; done'
```

如果停止后 Scene 正常打开，通常说明 UID 白名单漏放行了 Scene 的真实进程 UID。

检查日志和进程：

```sh
su -c 'cat /data/adb/modules/hideSceneport/hideport.log'
su -c 'ps -A -o USER,PID,PPID,NAME,ARGS | grep -Ei "scene|omarea|vtools|hideport"'
su -c 'ss -ltnp | grep -E "8765|8788"'
```

正常情况下，日志里的 `allowed uid` 应包含 Scene UI 进程对应的真实 UID。比如进程用户是 `u0_a384`，真实 UID 通常是 `10384`。

如果模块启动太早，旧版可能只解析到 `/data/data/com.omarea.vtools` 的 owner，比如 `999`，导致 Scene UI 自己被当成检测器拦截。新版已经改为等待包管理器或运行进程提供真实 UID 后再启动。

如果仍然漏 UID，可以在 `hideport.conf` 手动添加：

```sh
EXTRA_ALLOWED_UIDS="10384"
```

然后重启。

也要确认是否是自己手机导出的 `vmlinux.btf` 构建出来的包。

如果是从别人那里下载的预编译包，建议重新按上面的步骤自助构建。

### 这个模块会写 iptables 规则吗？

不会。当前版本不使用 `iptables` / `ip6tables`，也不再打包 `service.d` 端口隐藏脚本。

如果你看到 `8765`、`8788` 相关 iptables 规则，通常是旧版本残留或其他脚本写入。可以禁用旧模块并完整重启后再检查。

### 安装时报 Kernel BTF mismatch 是什么？

说明这个模块不是用当前手机的 `/sys/kernel/btf/vmlinux` 构建出来的。

请重新从当前手机导出 `vmlinux.btf`，上传到自己的 Fork，再跑一次 GitHub Actions。

## 捐赠

如果您觉得这个模块对您有帮助，可以考虑请作者喝杯咖啡：

特别感谢：

- 里（Luna Developers）
- 欣（Coolapk）

| 微信支付 | 支付宝 |
| :---: | :---: |
| ![微信支付](wx.png) | ![支付宝](zfb.jpg) |
