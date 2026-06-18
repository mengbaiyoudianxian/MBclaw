# 04 — 系统镜像集成方案

## 从应用到系统的升级

有系统镜像权限时，架构从"应用层打补丁"升级为"系统级原生集成"。

### 系统分区布局

```
/system/
├── app/MbclawSystem/          # priv-app 级别系统服务 APK
├── mbclaw/
│   ├── core/                  # FastAPI 全部代码
│   │   ├── app/
│   │   ├── data/mbclaw.db
│   │   └── requirements.txt
│   ├── python_runtime/        # 预装 CPython ARM64
│   ├── voice/                 # 离线语音模型
│   │   ├── wake_word_mbclaw.tflite
│   │   ├── asr_model/
│   │   └── tts_model/
│   ├── linux_sandbox/         # Ubuntu ARM64 rootfs
│   └── bin/                   # 系统级工具脚本
│       ├── mbclawd
│       ├── sandbox_ctl
│       └── sync_daemon
├── etc/init/mbclaw.rc         # Android init 启动脚本
└── etc/selinux/mbclaw.te      # SELinux 安全策略
```

### 关键对比：应用层 vs 系统级

| 模块 | 应用层 | 系统镜像 |
|------|--------|----------|
| MBclaw 进程 | APK 嵌入，可能被 LMK 杀掉 | init.rc 拉起守护进程，永驻后台 |
| Linux 沙盒 | proot 模拟（性能损耗 ~30%） | 原生 unshare + chroot，零损耗 |
| 语音唤醒 | 应用层拦截小爱（不稳定） | 直接替换 VoiceInteractionService |
| Python | Chaquopy / 嵌入方案 | 系统分区预装原生 CPython ARM64 |
| SELinux | 受限应用域 | 自定义 mbclaw_t 域，精确权限 |
| 开机启动 | BroadcastReceiver | class_start core 自动启动 |
| 系统接口 | 受限 API | system_api + signature|privileged 全权限 |

### 语音框架级替换

```
Audio HAL（DSP 低功耗监听）
  → VoiceInteractionManagerService
    → 原：分发到小爱同学
    → 改：分发到 MbclawVoiceService  ← 系统镜像修改点
      → 加载 /system/mbclaw/voice/
      → 唤醒词检测 → ASR → Agent → TTS
```

两种修改方式：
- **A：替换默认 VoiceInteractionService**（AOSP 镜像）
- **B：拦截 MIUI 小爱服务属性**（MIUI 镜像）

### 沙盒：proot → 原生容器

```
之前：用户指令 → proot 拦截系统调用 → 翻译 → 执行（损耗 30-40%）
现在：用户指令 → unshare 创建 namespace → 直接执行（损耗 < 1%）

unshare --mount --uts --ipc --net --pid --fork \
  chroot /system/mbclaw/linux_sandbox/... \
  /bin/bash -c "$COMMAND"
```
