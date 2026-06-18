# 03 — MBclaw × miclaw 融合架构方案

## 核心思路

以 miclaw 为 APK 包，借鉴 miclaw 中的智能体方案，把 MBclaw 部署在安卓 APK 环境里：

- 直接调用原生 miclaw 所有技能
- 使用系统镜像内的所有系统接口
- 云端沙盒改为本地沙盒（完整 Linux 环境支持），用来跑危险指令或安卓不支持的指令
- 原生支持将智能体部署在云服务器上，与本地连接无需 VPN

## 三阶段演进

```
Mode A（当前 MVP）      Mode B（中期）         Mode C（远期）
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│ Android APK   │      │ Android APK   │      │ Android+Linux│
│ + 本地沙盒    │ ───▶ │ + 云服务器    │ ───▶ │ 双系统原生    │
│（纯本地）     │      │（混合架构）   │      │（裸金属性能） │
└──────────────┘      └──────────────┘      └──────────────┘
```

## Mode A — 纯本地架构

### 分层结构

```
Android APK（miclaw）
├── UI Layer（保留 miclaw UI）
│   ├── 原 miclaw 功能界面
│   ├── MBclaw 对话界面
│   └── 语音对话界面（唤醒 + ASR + TTS）
├── Agent Core（MBclaw-Lite）
│   ├── FastAPI（嵌入式）
│   ├── Services（总结 / 关键词）
│   └── SQLite（本地存储）
├── miclaw Native Bridge
│   ├── 语音唤醒（替换小爱同学）
│   ├── 语音识别（ASR）
│   ├── TTS 语音合成
│   ├── 系统接口（全部 API）
│   ├── miclaw API Key（内测权限）
│   └── 传感器 / 通知 / 权限管理
└── 本地 Linux 沙盒（proot / Termux）
    └── 危险指令 / 安卓不支持的命令
```

### 语音交互流程

```
用户说"嘿 MBclaw"
  → 语音唤醒模块（替换小爱同学）
  → ASR 语音识别（5-6 秒无语音 → 自动发送）
  → MBclaw Agent Core 处理
    ├─ 安全指令 → Android 系统接口
    └─ 危险指令 → Linux 沙盒执行
  → TTS 语音合成 → 语音回复用户
  → 支持被打断（用户中途插话）
```

### 备份方案：云服务器

> 等后续备用机回来后，升级为原生安卓 + Linux 双系统

在备用机到位前，支持将智能体部署在云服务器上：
- 本地 miclaw APK ↔ 云服务器 MBclaw
- 无需 VPN（通过内网穿透或 WebSocket）
