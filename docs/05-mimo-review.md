# 05 — MiMo 视角架构纠错审查

> 基于小米技术栈实际情况的严格审查。

## 问题 1：/system 分区不可写

**当时方案**：把 MBclaw 放进 /system/mbclaw/，修改 init.rc

**现实**：
- Android 10-12 (MIUI 12-14): /system → ext4，可 remount rw（需 root）
- Android 13+ (HyperOS 1.0+): /system → erofs，只读，无法 remount rw

**结论**：Android 13+ / HyperOS 设备只能走**重新打包刷机**或**Magisk 模块 Magic Mount**

## 问题 2：init.rc 不可直接修改

**当时方案**：写 /system/etc/init/mbclaw.rc，用 class core 拉守护进程

**现实**：
- /system/etc/init/ 属于 system 镜像，erofs 不可写
- Xiaomi 的 init 会校验 vendor SELinux 策略，自定义 service 可能被阻断

**修正方案**：
- 方案 A: Magisk/KernelSU 模块 → /data/adb/modules/mbclaw/
- 方案 B: /data/local/tmp + nohup（最简单，但无开机自启）

## 问题 3：CONFIG_USER_NS 大概率未开启

**当时方案**：用 unshare 创建 Linux namespace 替代 proot

**现实**：
- 绝大多数 Xiaomi 设备 CONFIG_USER_NS=n
- Android 12+ 限制了非 root 进程创建 user namespace

**真实可行方案**：proot（虽然是性能妥协，但几乎是唯一可行方案）

## 问题 4：语音框架替换的真实限制

### DSP 固件是硬件瓶颈

```
DSP 唤醒词 → 硬编码 "小爱同学"
无法通过软件修改
硬件层级的唤醒 → 永远会触发小爱
```

### 修正方案

- **方案 A (推荐 MVP)**: 辅助唤醒 — 小爱同学保持不动，MBclaw 用应用层 VAD 检测。用户说"小爱同学，打开 MBclaw"
- **方案 B (实用)**: 快捷唤醒 — 长按电源键/锁屏左滑/通知栏快捷开关
- **方案 C (进阶)**: Magisk 替换 DSP 固件（高风险，不推荐 MVP）

## 问题 5：Python 在 Android 上的真实情况

**当时方案**：预装 CPython 到系统分区

**现实**：
- CPython 没有官方 Android 支持，需自己交叉编译 ARM64
- pip install 大量失败（numpy/pandas 需 Fortran，psutil 需 /proc）
- Chaquopy 仅支持 build.gradle 声明依赖，不支持运行时 pip install

**修正方案**：使用 Termux 作为 Python 运行时
- Termux 维护了完整的 Android Python 生态
- `pkg install python` → 直接可用，支持 pip install 几乎所有纯 Python 包
- APK ← localhost HTTP (127.0.0.1:8000) → Termux Python MBclaw Core

## 问题 6：后台保活

MIUI/HyperOS "神隐模式" 会杀后台进程

**推荐组合拳**：
- 前台服务 + 常驻通知（必须，否则秒杀）
- 引导用户关闭神隐模式 + 电池优化白名单
- 系统签名应用 + persistent=true（需系统镜像）
- 无障碍服务保活（次选）
