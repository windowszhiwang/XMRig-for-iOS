# XMRig iOS

把 [XMRig](https://github.com/xmrig/xmrig) 移植到越狱 iOS 设备的完整方案：在 iPhone/iPad 上挖门罗币（XMR），含控制 App 与后台保活服务。

此项目使用muse.ai构建，没有在多数新款设备上进行测试  如有问题，可以自己使用muse进行vide coding（没有额度了）

如果你想要在Windows电脑上面进行运行，那么你可以下载更优秀的一键运行exe  https://www.kryptex.com/?ref=e9dbbc79

## 功能

- **XMRig App**（`com.user.xmrig-app`）：原生 UIKit 应用，桌面图标
  - 配置矿池地址、矿工名、密码、线程数
  - 轻量模式开关（老设备必备）
  - 一键启动 / 停止，后台进程状态检测
  - 实时查看矿机日志、内置诊断工具
- **保活服务**（`com.user.xmrig-daemon`）：无界面 Cydia 插件
  - LaunchDaemon 开机自启，崩溃自动拉起
  - 以 root 身份、`Nice -20` 最高优先级运行
- **独立 IPA**：也可单独分发安装

## 系统要求

- 已越狱的 iPhone/iPad（arm64）
- iOS 12.0+
- 内存 ≥ 1GB
- 可访问矿池的网络（大陆地区需 VPN）

## 安装

### 方式一：Cydia 源（推荐）

1. Cydia 添加源：`http://你的服务器IP/`
2. 安装 `XMRig App` 和 `XMRig Daemon (保活)`
3. 桌面打开 XMRig，填好矿池信息点保存 → 启动

### 方式二：手动安装

- `XMRig.ipa`：通过 AppSync / TrollStore（iOS 14+）安装，或解压后将 `XMRig.app` 拷入 `/Applications/` 并执行 `uicache`
- `.deb` 包：`dpkg -i` 安装

## 配置说明

| 选项 | 说明 | 默认值 |
|------|------|--------|
| 矿池地址 | `host:port`，仅支持非 TLS 端口 | `xmr-hk.kryptex.network:7029` |
| 矿工名 | 任意标识 | `ios-miner` |
| 矿池密码 | 一般填 `x` | `x` |
| 线程数 | 留空=自动，老机器建议 1-2 | `2` |
| 轻量模式 | RandomX light 模式，1GB 内存设备必须开 | 开 |

配置文件位置：`/var/mobile/Library/Preferences/com.user.xmrigsettings.plist`
矿机日志：`/var/log/xmrig.log`

## 技术细节

### 交叉编译

在 Linux 上用 clang 18 + lld + iPhoneOS SDK 交叉编译 XMRig 6.26.0（CPU only，关闭 TLS/OpenCL/CUDA/hwloc）：

```bash
-target arm64-apple-ios -miphoneos-version-min=12.0 -fuse-ld=lld
```

### iOS 适配补丁

1. **`Platform_mac.cpp`**：绕开 iOS 不存在的 macOS IOKit 电源管理接口
2. **`VirtualMemory_unix.cpp`**：用 `dlsym` 动态解析 `pthread_jit_write_protect_np`（SDK 标记为不可用，但真机存在）
3. **`RxVm.cpp`**：iOS 跳过 `RANDOMX_FLAG_JIT`，强制解释器模式（iOS 禁止 RWX 内存，JIT 无法工作）
4. **`xmrig.cpp`**：`main()` 开头加 `setvbuf(_IONBF)`，日志实时落盘（重定向到文件时 libc 默认 4KB 缓冲）
5. **`xmrig.cpp`**：全局 `set_terminate` 钩子，便于抓取崩溃时的异常信息

### 打包要点

- `.deb` 必须用 **gzip** 压缩（`dpkg-deb -Zgzip`），新版 dpkg 默认的 zstd 老设备解不开
- App 二进制需要 `platform-application` + `com.apple.private.security.no-sandbox` entitlements，否则无法 `posix_spawn` 子进程
- App 以 setuid root（6755）安装，以便管理 LaunchDaemon
- 启动器为 C 二进制（读 plist → 组装参数 → `exec xmrig`），不依赖 shell（越狱机常缺 coreutils）
- 进程检测、日志读取全部用原生 API（sysctl / NSFileManager），不依赖 `ps`/`grep`/`tail`

## 已知限制

- 解释器模式算力约为 JIT 模式的 1/5~1/10，iPhone 6 实测仅适合体验
- 编译时关闭了 TLS，矿池必须用非 TLS 端口
- 未在多种越狱环境下充分测试，以实际运行为准

## 文件结构

```
├── xmrig/                  # XMRig 源码（含 iOS 补丁）
├── daemon/                 # C 语言启动器源码
├── app/                    # iOS App 源码（单文件 UIKit）
├── daemon-pkg/             # 保活包打包目录
├── app-pkg/                # App 包打包目录
├── output/                 # 成品（.deb / .ipa / Packages 索引）
│   └── repo/               # Cydia 仓库文件
└── sdks/                   # iPhoneOS SDK
```

## 免责声明

本项目仅供学习研究。挖矿会显著增加设备发热、耗电并加速硬件老化，请自行评估风险。遵守当地法律法规。
