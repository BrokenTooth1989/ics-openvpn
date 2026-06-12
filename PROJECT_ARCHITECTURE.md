# ics-openvpn 项目架构详细文档

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 设计原理](#2-设计原理)
  - [2.1 双进程架构](#21-双进程架构)
  - [2.2 事件驱动与观察者模式](#22-事件驱动与观察者模式)
  - [2.3 AIDL 跨进程通信设计](#23-aidl-跨进程通信设计)
  - [2.4 管理接口（Management Interface）设计](#24-管理接口management-interface设计)
    - [2.4.1 架构概述](#241-架构概述)
    - [2.4.2 Socket 创建与连接流程](#242-socket-创建与连接流程)
    - [2.4.3 管理接口协议](#243-管理接口协议)
    - [2.4.4 文件描述符传递机制（SCM_RIGHTS）](#244-文件描述符传递机制scm_rights)
    - [2.4.5 关键源文件索引](#245-关键源文件索引)
    - [2.4.6 management_io 触发机制](#246-management_io-触发机制)
  - [2.5 前台服务与生命周期管理](#25-前台服务与生命周期管理)
- [3. 项目目录结构](#3-项目目录结构)
- [4. 核心类详解](#4-核心类详解)
  - [4.1 VpnStatus — 全局状态中心](#41-vpnstatus--全局状态中心)
  - [4.2 OpenVPNService — VPN 核心服务](#42-openvpnservice--vpn-核心服务)
  - [4.3 OpenVPNStatusService — 状态桥接服务](#43-openvpnstatusservice--状态桥接服务)
  - [4.4 OpenVpnManagementThread — 管理接口线程](#44-openvpnmanagementthread--管理接口线程)
  - [4.5 VPNLaunchHelper — 启动辅助类](#55-vpnlaunchhelper--启动辅助类)
  - [4.6 VpnProfile — 配置模型](#46-vpnprofile--配置模型)
  - [4.7 ProfileManager — 配置管理器](#47-profilemanager--配置管理器)
  - [4.8 StatusListener — UI 进程状态监听器](#48-statuslistener--ui-进程状态监听器)
  - [4.9 DeviceStateReceiver — 设备状态监听](#49-devicestatereceiver--设备状态监听)
  - [4.10 keepVPNAlive — 保活机制](#410-keepvpnalive--保活机制)
- [5. AIDL 接口详解](#5-aidl-接口详解)
  - [5.1 内部 AIDL 接口](#51-内部-aidl-接口)
  - [5.2 外部 API AIDL 接口](#52-外部-api-aidl-接口)
  - [5.3 外部证书提供者 AIDL](#53-外部证书提供者-aidl)
  - [5.4 Parcelable 数据类](#54-parcelable-数据类)
- [6. Service 详解及与 AIDL 的关系](#6-service-详解及与-aidl-的关系)
  - [6.1 OpenVPNService](#61-openvpnservice)
  - [6.2 OpenVPNStatusService](#62-openvpnstatusservice)
  - [6.3 ExternalOpenVPNService](#63-externalopenvpnservice)
  - [6.4 keepVPNAlive](#64-keepvpnalive)
  - [6.5 Service 与 AIDL 绑定关系总览](#65-service-与-aidl-绑定关系总览)
- [7. 完整调用流程](#7-完整调用流程)
  - [7.1 用户发起 VPN 连接](#71-用户发起-vpn-连接)
  - [7.2 OpenVPNService 启动与配置](#72-openvpnservice-启动与配置)
  - [7.3 原生 OpenVPN 进程启动与管理接口交互](#73-原生-openvpn-进程启动与管理接口交互)
  - [7.4 TUN 设备建立](#74-tun-设备建立)
  - [7.5 状态传播机制](#75-状态传播机制)
  - [7.6 VPN 断开流程](#76-vpn-断开流程)
  - [7.7 外部 App 控制 VPN 流程](#77-外部-app-控制-vpn-流程)
  - [7.8 开机自启动流程](#78-开机自启动流程)
- [8. 进程间通信架构图](#8-进程间通信架构图)
- [9. 状态机与连接状态](#9-状态机与连接状态)
- [10. 关键设计决策与权衡](#10-关键设计决策与权衡)
- [11. 代码解析](#11-代码解析)
  - [11.1 SignaturePadding 枚举 — 外部 PKI 签名填充模式](#111-signaturepadding-枚举--外部-pki-签名填充模式)
  - [11.2 OpenVPN 2.x 与 OpenVPN 3.x 双引擎架构](#112-openvpn-2x-与-openvpn-3x-双引擎架构)
    - [11.2.1 架构选择机制](#1121-架构选择机制)
    - [11.2.2 核心架构对比](#1122-核心架构对比)
    - [11.2.3 OpenVPN 2.x 详细架构](#1123-openvpn-2x-详细架构)
    - [11.2.4 OpenVPN 3.x 详细架构](#1124-openvpn-3x-详细架构)
    - [11.2.5 OpenVPNManagement 接口的两种实现](#1125-openvpnmanagement-接口的两种实现)
    - [11.2.6 OpenVPN 3.x 的限制](#1126-openvpn-3x-的限制)
    - [11.2.7 完整对比图](#1127-完整对比图)
    - [11.2.8 关键源文件索引](#1128-关键源文件索引)

---

## 1. 项目概述

ics-openvpn 是一个基于 OpenVPN 的 Android VPN 客户端应用。它通过 Android 的 `VpnService` API 创建虚拟网络接口（TUN 设备），将系统网络流量路由到该接口，然后由原生的 OpenVPN 进程（C/C++ 实现）对流量进行加密并传输到远程 VPN 服务器。

项目核心特点：
- **双进程架构**：UI 进程和 VPN 引擎进程（`:openvpn`）分离
- **AIDL 跨进程通信**：通过多组 AIDL 接口实现进程间状态同步和控制
- **外部 API**：允许第三方应用通过 AIDL 接口控制 VPN 连接
- **管理接口**：通过 Unix Domain Socket 与原生 OpenVPN 进程通信
- **前台服务**：VPN 以 Android 前台服务运行，保证系统不会轻易杀死

---

## 2. 设计原理

### 2.1 双进程架构

应用运行在两个进程中：

| 进程 | 名称 | 包含组件 | 职责 |
|------|------|----------|------|
| **UI 进程** | 默认进程（包名） | MainActivity, LogWindow, VPNPreferences, StatusListener, LaunchVPN, Fragments, OpenVPNTileService | 用户界面、配置文件管理、设置、日志显示 |
| **:openvpn 进程** | `:openvpn` | OpenVPNService, OpenVPNStatusService, ExternalOpenVPNService, keepVPNAlive | VPN 引擎运行、原生进程管理、状态广播、外部 API 服务 |

**为什么需要双进程？**

1. **稳定性隔离**：VPN 服务需要长时间稳定运行，UI 进程可能因用户操作或内存压力被系统回收，但 VPN 连接不能中断
2. **权限隔离**：`VpnService` 需要 `BIND_VPN_SERVICE` 权限，在独立进程中更容易管理
3. **生命周期独立**：VPN 前台服务的生命周期与 Activity 完全独立，避免 Activity 销毁导致 VPN 断开
4. **外部 API 隔离**：第三方应用通过 AIDL 绑定到 `:openvpn` 进程，不会影响 UI 进程

AndroidManifest.xml 中的声明：
```xml
<service android:name=".core.OpenVPNService"
         android:process=":openvpn"
         android:permission="android.permission.BIND_VPN_SERVICE" />

<service android:name=".core.OpenVPNStatusService"
         android:process=":openvpn"
         android:exported="false" />

<service android:name=".api.ExternalOpenVPNService"
         android:process=":openvpn"
         android:exported="true" />
```

### 2.2 事件驱动与观察者模式

项目使用 **VpnStatus** 作为全局事件总线（单例模式），实现观察者模式：

```
VpnStatus (全局单例，每个进程各一个实例)
├── LogListener       → 日志事件
├── StateListener     → 连接状态事件
├── ByteCountListener → 流量统计事件
└── ProfileNotifyListener → 配置文件变更事件
```

**在 `:openvpn` 进程中**，VpnStatus 的监听器包括：
- `OpenVPNService` — 更新通知栏、广播 VPN 状态
- `OpenVPNStatusService` — 通过 AIDL 转发状态到 UI 进程
- `ExternalOpenVPNService` — 通过 AIDL 转发状态到外部应用

**在 UI 进程中**，VpnStatus 的监听器包括：
- `StatusListener`（从 AIDL 接收事件后注入 VpnStatus）
- 各种 UI 组件（LogFragment, GraphFragment 等）

关键代码流程（`VpnStatus.java:389-409`）：
```java
public synchronized static void updateStateString(String state, String msg, int resid, 
                                                   ConnectionStatus level, Intent intent) {
    mLaststate = state;
    mLaststatemsg = msg;
    mLastStateresid = resid;
    mLastLevel = level;
    mLastIntent = intent;

    for (StateListener sl : stateListener) {
        sl.updateState(state, msg, resid, level, intent);
    }
}
```

### 2.3 AIDL 跨进程通信设计

项目定义了 **三组 AIDL 接口**，分别服务于不同的通信场景：

#### 第一组：内部状态桥接（`:openvpn` → UI）

| 接口 | 方向 | 用途 |
|------|------|------|
| `IServiceStatus` | UI → `:openvpn` | 注册/注销回调、获取最近连接的 VPN、设置缓存密码、获取流量历史 |
| `IStatusCallbacks` | `:openvpn` → UI | 回调：新日志、状态更新、流量统计、已连接 VPN UUID、配置版本变更 |

这一组用于 `OpenVPNStatusService` 向 UI 进程的 `StatusListener` 传递实时事件。

#### 第二组：内部 VPN 控制（UI/外部 → `:openvpn`）

| 接口 | 方向 | 用途 |
|------|------|------|
| `IOpenVPNServiceInternal` | 外部 → `:openvpn` | protect socket FD、暂停/恢复 VPN、停止 VPN、管理外部应用权限、Challenge-Response |

这一组用于控制 `OpenVPNService` 的运行行为。

#### 第三组：外部应用 API（第三方 App → `:openvpn`）

| 接口 | 方向 | 用途 |
|------|------|------|
| `IOpenVPNAPIService` | 第三方 App → `:openvpn` | 获取配置列表、启动/停止 VPN、添加/删除配置、保护 Socket、注册状态回调 |
| `IOpenVPNStatusCallback` | `:openvpn` → 第三方 App | 回调：连接状态变更通知 |
| `ExternalCertificateProvider` | `:openvpn` → 第三方 App | 外部证书签名、证书链获取、证书元数据 |

### 2.4 管理接口（Management Interface）设计

Java 层与原生 OpenVPN 进程之间通过 **Unix Domain Socket (UDS)** 通信。这是一个基于文本的、行分隔的协议，运行在 OpenVPN 标准 Management Interface 之上，并扩展了 Android 特有的命令（如 `PROTECTFD`、`OPENTUN`）。

#### 2.4.1 架构概述

**Java 层是 UDS 服务端（监听者），原生 OpenVPN 进程是客户端（连接者）。** 这与通常的 OpenVPN Management Interface（原生进程作为服务端）相反，原因是 Java 层需要调用 `VpnService.protect()` 和 `VpnService.Builder.establish()` 等 Android 专属 API，这些 API 只能在 Java 进程中执行。

```
┌───────────────────────────────────┐       Unix Domain Socket        ┌──────────────────┐
│  OpenVpnManagementThread          │◄────────────────────────────────►│  OpenVPN 进程     │
│  (Java, :openvpn 进程, 服务端)     │   cacheDir/mgmtsocket           │  (C/C++, 客户端)  │
│                                   │   (文件系统路径)                  │                  │
│  创建 LocalServerSocket           │                                  │  connect() 连接   │
│  accept() 等待连接                 │                                  │  管理 CLI 交互    │
│  通过 SCM_RIGHTS 收/发 FD         │                                  │  通过 SCM_RIGHTS  │
│  调用 VpnService API              │                                  │   收/发 FD        │
└───────────────────────────────────┘                                  └──────────────────┘
```

#### 2.4.2 Socket 创建与连接流程

**Step 1: Java 端配置管理接口参数**（`VpnProfile.java:407-415`）

在生成 OpenVPN 配置文本时，写入以下指令：

```
management <cacheDir>/mgmtsocket unix
management-client
management-query-passwords
management-hold
```

- `management ... unix`：指定使用 Unix Domain Socket，路径为 `cacheDir/mgmtsocket`
- `management-client`：让 OpenVPN 进程以客户端模式连接（而非监听模式）
- `management-query-passwords`：通过管理接口请求密码
- `management-hold`：启动时进入 hold 状态，等待 Java 端发送 `hold release`

**Step 2: Java 端创建 UDS 服务端**（`OpenVpnManagementThread.java:117-150`）

```java
// 创建 socket 路径
String socketName = cacheDir.getAbsolutePath() + "/mgmtsocket";

// 创建并绑定 LocalSocket
mServerSocketLocal = new LocalSocket();
mServerSocketLocal.bind(new LocalSocketAddress(socketName, LocalSocketAddress.Namespace.FILESYSTEM));

// 包装为 LocalServerSocket 并开始监听
mServerSocket = new LocalServerSocket(mServerSocketLocal.getFileDescriptor());
```

**Step 3: Java 端等待原生进程连接**（`OpenVpnManagementThread.java:181`）

```java
mSocket = mServerSocket.accept();  // 阻塞等待
mSocket.getInputStream();           // 获取输入流
```

**Step 4: 原生端解析配置并连接**（C 层）

| 文件 | 行号 | 函数 | 动作 |
|------|------|------|------|
| `options.c` | 5706-5717 | 解析 `--management <addr> unix` | 设置 `MF_UNIX_SOCK` 标志，存储 socket 路径 |
| `manage.c` | 2631-2667 | `man_settings_init()` | 初始化 `sockaddr_un` 结构体 |
| `manage.c` | 2764-2771 | `man_connection_init()` | 检测到 `MF_CONNECT_AS_CLIENT`，调用 `man_connect()` |
| `manage.c` | 2101-2127 | `man_connect()` | 调用 `create_socket_unix()` + `socket_connect_unix()` |
| `socket.c` | 3009-3022 | `create_socket_unix()` | `socket(PF_UNIX, SOCK_STREAM, 0)` |
| `socket.c` | 3059-3066 | `socket_connect_unix()` | `connect()` 到 Java 端的 socket 路径 |

**Step 5: Java 端发送初始命令**（`OpenVpnManagementThread.java:194-195`）

连接建立后，Java 端发送：

```
version 3          // 协商管理接口版本 3
hold release       // 释放 hold 状态，让 OpenVPN 开始连接
bytecount 2        // 每 2 秒报告一次流量统计
state on           // 启用实时状态通知
```

**完整时序图：**

```
Java (UDS 服务端)                               OpenVPN 进程 (UDS 客户端)
========================                        =========================
openManagementInterface()
  bind("cacheDir/mgmtsocket")
  listen()
  accept() [阻塞等待]
                                                进程启动，读取配置:
                                                  management <path>/mgmtsocket unix
                                                  management-client
                                                man_connect()
                                                  socket(PF_UNIX, SOCK_STREAM, 0)
                                                  connect("<path>/mgmtsocket")
  accept() 返回 mSocket
  发送: "version 3\n"
                                                接收: "version 3\n"
                                                发送: ">INFO:..."
  发送: "hold release\n"
  发送: "state on\n"
  发送: "bytecount 2\n"
                                                开始正常 VPN 连接流程
```

#### 2.4.3 管理接口协议

协议是 **文本行协议**，每条消息以 `\n` 结尾。

**原生 → Java（异步通知）：**

| 消息格式 | 含义 | Java 端处理 |
|----------|------|-------------|
| `>PASSWORD:Need 'Auth' password` | 请求认证凭据 | 从 PasswordCache 获取用户名/密码 |
| `>PASSWORD:Verification Failed: 'Auth'` | 认证失败 | 更新状态为 AUTH_FAILED |
| `>HOLD:Waiting for hold release:N` | 进程处于 hold 状态 | 延迟后自动 releaseHold |
| `>NEED-OK:OPENTUN:MSG:tun` | 请求创建 TUN 设备 | 调用 `openTun()`，通过 FD 传递返回 |
| `>NEED-OK:DNSSERVER:...` | 配置 DNS 服务器 | 添加到 tunConfig.mDnslist |
| `>NEED-OK:ROUTE:...` | 配置路由 | 添加到 tunConfig.mRoutes |
| `>NEED-OK:ROUTE6:...` | 配置 IPv6 路由 | 添加到 tunConfig.mRoutes |
| `>NEED-OK:IFCONFIG:...` | 配置本地 IP | 设置 tunConfig.setLocalIP() |
| `>NEED-OK:IFCONFIG6:...` | 配置 IPv6 地址 | 设置 IPv6 地址 |
| `>NEED-OK:HTTPPROXY:...` | 配置 HTTP 代理 | 设置代理信息 |
| `>NEED-OK:PERSIST_TUN_ACTION:...` | 询问 TUN 持久化行为 | 返回 ok 或 cancel |
| `>BYTECOUNT:in,out` | 流量统计 | `VpnStatus.updateByteCount()` |
| `>STATE:timestamp,state,info,...` | 连接状态变化 | `VpnStatus.updateStateString()` |
| `>PROXY:connnum,proto` | 代理信息请求 | ProxyDetection 检测代理 |
| `>LOG:timestamp,level,ovpnlevel,message` | 日志消息 | `VpnStatus.logMessageOpenVPN()` |
| `>PK_SIGN:data,padding,...` | 外部密钥签名请求 | ExtAuthHelper 调用外部证书 App |
| `>INFOMSG:OPEN_URL:url` | SSO 浏览器认证 | `OpenVPNService.trigger_sso()` |
| `>INFOMSG:CR_TEXT:text` | 挑战-响应 | `OpenVPNService.trigger_sso()` |
| `>INFOMSG:WEB_AUTH:url` | 网页认证 | `OpenVPNService.trigger_sso()` |
| `PROTECTFD: fd 'N' sent to be protected` | 请求保护 FD | `VpnService.protect(fdint)` |
| `SUCCESS: ...` / `ERROR: ...` | 命令确认 | 确认或记录错误 |

**Java → 原生（控制命令）：**

| 命令 | 含义 | 发送位置 |
|------|------|----------|
| `version 3\n` | 设置管理接口版本 | 连接建立后立即发送 |
| `hold release\n` | 释放 hold 状态 | 连接建立后 / reconnect 后 |
| `state on\n` | 启用状态通知 | 连接建立后 |
| `bytecount N\n` | 启用流量统计（每 N 秒） | 连接建立后 |
| `username 'TYPE' VALUE\n` | 提供用户名 | 响应 `>PASSWORD:Need` |
| `password 'TYPE' VALUE\n` | 提供密码 | 响应 `>PASSWORD:Need` |
| `needok 'TYPE' ok\n` | 确认 NEED-OK 请求 | 响应 `>NEED-OK:` |
| `needok 'TYPE' cancel\n` | 取消 NEED-OK 请求 | 响应 `>NEED-OK:` |
| `proxy TYPE HOST PORT\n` | 设置代理 | 响应 `>PROXY:` |
| `proxy NONE\n` | 不使用代理 | 响应 `>PROXY:` |
| `network-change\n` | 通知网络变化 | 网络切换时 |
| `network-change samenetwork\n` | 通知同网络变化 | 网络切换时 |
| `signal SIGINT\n` | 停止 OpenVPN 进程 | 断开 VPN 时 |
| `signal SIGUSR1\n` | 重启 OpenVPN 进程 | reconnect 时 |
| `cr-response RESPONSE\n` | 挑战-响应回复 | 响应 CR_TEXT |
| `pk-sig\n` + data + `\nEND\n` | 外部密钥签名结果 | 响应 `>PK_SIGN:` |

#### 2.4.4 文件描述符传递机制（SCM_RIGHTS）

Unix Domain Socket 支持通过 **ancillary data（辅助数据）** 传递文件描述符，这使用 `SCM_RIGHTS` 机制。项目中有两个关键的 FD 传递场景：

##### 场景一：PROTECTFD（原生 → Java，保护 Socket）

OpenVPN 创建真实的网络 socket 后，需要调用 `VpnService.protect()` 防止 VPN 流量回环。但 `protect()` 是 Java API，原生进程无法直接调用。

**完整流程：**

```
1. 原生 OpenVPN 创建 socket fd
2. protect_fd_nonlocal() (socket.c:733) 判断地址非本地后进入保护流程
3. 设置 management->connection.fdtosend = fd  (socket.c:750)
4. 调用 management_android_control(management, "PROTECTFD", __func__) (socket.c:751)
5. management_android_control() (manage.c:2371) 调用 management_query_user_pass()
   以 GET_USER_PASS_NEED_OK 模式发起查询
6. management_query_user_pass() (manage.c:3610) 生成管理接口消息：
   ">NEED-OK:Need 'PROTECTFD' confirmation MSG:protect_fd_nonlocal"
   通过 msg(M_CLIENT, ...) 写入输出缓冲区 (manage.c:3690)
7. man_write() (manage.c:2515) 从输出缓冲区取数据发送时：
   检测到 man->connection.fdtosend > 0 (manage.c:2527)
8. 调用 man_send_with_fd(man->connection.sd_cli, BPTR(buf), len, MSG_NOSIGNAL,
                         man->connection.fdtosend) (manage.c:2529)
   → sendmsg() + SCM_RIGHTS 将 FD 作为 ancillary data 随文本一起发送
9. 发送后将 fdtosend 重置为 -1 (manage.c:2531)

── Java 端接收 ──

10. 主读取循环 (OpenVpnManagementThread.java:198-216)
    instream.read(buffer) 读取文本数据
11. mSocket.getAncillaryFileDescriptors() (OpenVpnManagementThread.java:204)
    从 Unix Domain Socket 接收 SCM_RIGHTS 传递的 FD
    → Collections.addAll(mFDList, fds) 存入队列 (OpenVpnManagementThread.java:209)

── Java 端处理（两条路径） ──

路径A: processCommand 处理 "PROTECTFD: ..." 文本行
12a. processInput() 解析出以 "PROTECTFD: " 开头的行 (OpenVpnManagementThread.java:321)
13a. mFDList.pollFirst() 取出 FileDescriptor (OpenVpnManagementThread.java:322)
14a. protectFileDescriptor(fdtoprotect) (OpenVpnManagementThread.java:324)

路径B: processNeedCommand 处理 ">NEED-OK:'PROTECTFD'..." 响应
12b. processNeedCommand() 中 case "PROTECTFD" (OpenVpnManagementThread.java:541)
13b. mFDList.pollFirst() 取出 FileDescriptor (OpenVpnManagementThread.java:542)
14b. protectFileDescriptor(fdtoprotect) (OpenVpnManagementThread.java:543)

── 最终保护 ──

15. protectFileDescriptor() (OpenVpnManagementThread.java:228)
    通过反射 FileDescriptor.getInt$() 获取 int 值 (OpenVpnManagementThread.java:230-231)
16. mOpenVPNService.protect(fdint) 调用 VpnService.protect() (OpenVpnManagementThread.java:235)
17. fdClose(fd) 关闭 FileDescriptor (OpenVpnManagementThread.java:239)
18. Java 端回复 "needok 'PROTECTFD' ok\n" → 原生端继续执行
```

**原生端发送 FD（`manage.c:2285-2315`）：**

`man_send_with_fd()` 使用 `sendmsg()` + `SCM_RIGHTS` 将 FD 作为 ancillary data 发送：

```c
static ssize_t
man_send_with_fd(int fd, void *ptr, size_t nbytes, int flags, int sendfd)
{
    struct msghdr msg = { 0 };
    struct iovec iov[1];

    union
    {
        struct cmsghdr cm;
        char control[CMSG_SPACE(sizeof(int))];
    } control_un;
    struct cmsghdr *cmptr;

    msg.msg_control = control_un.control;
    msg.msg_controllen = sizeof(control_un.control);

    cmptr = CMSG_FIRSTHDR(&msg);
    cmptr->cmsg_len = CMSG_LEN(sizeof(int));
    cmptr->cmsg_level = SOL_SOCKET;
    cmptr->cmsg_type = SCM_RIGHTS;
    *((int *)CMSG_DATA(cmptr)) = sendfd;    // 将 FD 放入 ancillary data

    msg.msg_name = NULL;
    msg.msg_namelen = 0;

    iov[0].iov_base = ptr;                   // 文本数据（包含 PROTECTFD 信息）
    iov[0].iov_len = nbytes;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    return (sendmsg(fd, &msg, flags));       // 文本 + FD 一起发送
}
```

**原生端触发发送（`manage.c:2514-2557`）：**

`man_write()` 在写出缓冲区数据时，检测到 `fdtosend > 0` 则调用 `man_send_with_fd()`：

```c
static ssize_t
man_write(struct management *man)
{
    const int size_hint = 1024;
    ssize_t sent = 0;
    const struct buffer *buf;

    buffer_list_aggregate(man->connection.out, size_hint);
    buf = buffer_list_peek(man->connection.out);
    if (buf && BLEN(buf))
    {
        const int len = min_int(size_hint, BLEN(buf));
#ifdef TARGET_ANDROID
        if (man->connection.fdtosend > 0)
        {
            sent = man_send_with_fd(man->connection.sd_cli, BPTR(buf), len, MSG_NOSIGNAL,
                                    man->connection.fdtosend);
            man->connection.fdtosend = -1;   // 发送后重置
        }
        else
#endif
        {
            sent = send(man->connection.sd_cli, (const void *)BPTR(buf), len, MSG_NOSIGNAL);
        }
        // ... 错误处理 ...
    }
    // ...
}
```

**原生端 `msg(M_CLIENT)` 与 `sendmsg()` 的分工：**

`msg(M_CLIENT, "...")` 和 `sendmsg()` 都参与了数据发送，但角色不同——前者是**生产者**（写入缓冲区），后者是**消费者**（发送到 socket）。

`msg(M_CLIENT)` 不直接发送 socket 数据，而是将文本写入输出缓冲区，调用链如下：

```
msg(M_CLIENT, ">NEED-OK:Need 'PROTECTFD' confirmation MSG:...")
  → virtual_output_callback_func() (manage.c:360)
    → flags == M_CLIENT
      → log_entry_print() 格式化为带 CRLF 的字符串
      → man_output_list_push_str(man, out) (manage.c:407)
        → buffer_list_push(man->connection.out, str) (manage.c:288)  ← 写入输出缓冲区
```

`sendmsg()` / `send()` 才是真正通过 Unix Domain Socket 将数据发送给 Java 端的系统调用：

```
management_io()
  → man_write() (manage.c:2515)
    → buffer_list_peek(man->connection.out)  从输出缓冲区取数据
    → fdtosend > 0 ?
        → man_send_with_fd() → sendmsg()  (附带 SCM_RIGHTS FD)
        → send()                            (普通发送，无 FD)
```

两者职责对比：

| 函数 | 角色 | 操作 | 是否接触 socket |
|------|------|------|-----------------|
| `msg(M_CLIENT, ...)` | 生产者 | 格式化文本 → 写入 `man->connection.out` 缓冲区 | 否 |
| `sendmsg()` / `send()` | 消费者 | 从缓冲区取数据 → 系统调用发送到 socket | 是（系统调用） |

对于 PROTECTFD 场景，FD 只能通过 `sendmsg()` + `SCM_RIGHTS` 传递（`send()` 不支持 ancillary data），所以 `man_write()` 检测到 `fdtosend > 0` 时走 `man_send_with_fd()` 分支而非普通 `send()`。

**`man_send_with_fd()` 与 Java 端 `case "PROTECTFD"` 的关联机制：**

原生端通过 `sendmsg()` 在**一次系统调用**中原子地同时发送文本数据和 FD，Java 端在**同一次 `read()` 调用**中同时收到两者。两者靠时序关联，而非 FD 编号。

原生端发送（一次 `sendmsg()`，文本 + FD 同时发出）：

```
man_send_with_fd() 调用 sendmsg()，消息包含两部分：
  ├── iov[0] (普通数据): 文本，如 ">NEED-OK:Need 'PROTECTFD' confirmation MSG:protect_fd_nonlocal\n"
  │                      或 "PROTECTFD: fd '5' sent to be protected\n"
  └── cmsg (辅助数据):  FD（通过 SCM_RIGHTS）
```

Java 端接收（同一次循环迭代中同时拿到文本和 FD）：

```java
// OpenVpnManagementThread.java:198-216 主读取循环
while (true) {
    int numbytesread = instream.read(buffer);             // ① 读到文本数据
    FileDescriptor[] fds = mSocket.getAncillaryFileDescriptors(); // ② 拿到伴随而来的 FD
    if (fds != null) {
        Collections.addAll(mFDList, fds);                 // ③ FD 入队
    }
    pendingInput = processInput(pendingInput);             // ④ 解析文本 → 触发 FD 出队
}
```

文本解析触发 FD 消费（两条路径）：

```
路径A: 文本 "PROTECTFD: fd '5' sent to be protected\n"
  → processCommand() 匹配 command.startsWith("PROTECTFD: ")  (321行)
  → mFDList.pollFirst() 取出 FD  →  protectFileDescriptor()

路径B: 文本 ">NEED-OK:Need 'PROTECTFD' confirmation MSG:protect_fd_nonlocal\n"
  → processCommand() 匹配 cmd == "NEED-OK"  (291行)
  → processNeedCommand() → case "PROTECTFD"  (541行)
  → mFDList.pollFirst() 取出 FD  →  protectFileDescriptor()
```

关键点：

- **不是靠 FD 编号关联**：文本中的 `"fd '5'"` 仅是日志信息，Java 端不会解析这个数字，而是直接从 `mFDList` 队列按先进先出取出
- **靠时序关联**：`sendmsg()` 保证文本和 FD 在同一条 socket 消息中原子发送，Java 端在同一次 `read()` 中同时收到，FD 先入队，文本解析时立即出队消费
- **队列为空则丢弃**：如果 `mFDList.pollFirst()` 返回 `null`（路径A 322行有检查），则跳过保护操作

**原生端保护流程入口（`socket.c:732-752`）：**

```c
#ifdef TARGET_ANDROID
static void
protect_fd_nonlocal(int fd, const struct sockaddr *addr)
{
    if (!management)
    {
        msg(M_FATAL, "Required management interface not available.");
    }

    if (addr_local(addr))
    {
        msg(D_SOCKET_DEBUG, "Address is local, not protecting socket fd %d", fd);
        return;
    }

    msg(D_SOCKET_DEBUG, "Protecting socket fd %d", fd);
    management->connection.fdtosend = fd;    // 设置待发送的 FD
    management_android_control(management, "PROTECTFD", __func__);
}
#endif
```

**Java 端接收并保护 FD（`OpenVpnManagementThread.java`）：**

主读取循环中接收 ancillary FD：

```java
// 196-216: 主读取循环
while (true) {
    int numbytesread = instream.read(buffer);
    if (numbytesread == -1)
        return;

    FileDescriptor[] fds = null;
    try {
        fds = mSocket.getAncillaryFileDescriptors();  // 接收 SCM_RIGHTS FD
    } catch (IOException e) {
        VpnStatus.logException("Error reading fds from socket", e);
    }
    if (fds != null) {
        Collections.addAll(mFDList, fds);             // 存入 FD 队列
    }

    String input = new String(buffer, 0, numbytesread, "UTF-8");
    pendingInput += input;
    pendingInput = processInput(pendingInput);
}
```

路径A — 处理 `PROTECTFD:` 文本行：

```java
// 321-324: processCommand 中处理 PROTECTFD
} else if (command.startsWith("PROTECTFD: ")) {
    FileDescriptor fdtoprotect = mFDList.pollFirst();  // 从队列取出 FD
    if (fdtoprotect != null)
        protectFileDescriptor(fdtoprotect);
}
```

路径B — 处理 `NEED-OK:'PROTECTFD'` 响应：

```java
// 541-543: processNeedCommand 中处理 PROTECTFD
case "PROTECTFD":
    FileDescriptor fdtoprotect = mFDList.pollFirst();  // 从队列取出 FD
    protectFileDescriptor(fdtoprotect);
    break;
```

保护 FD 的实现：

```java
// 228-239: protectFileDescriptor
private void protectFileDescriptor(FileDescriptor fd) {
    try {
        Method getInt = FileDescriptor.class.getDeclaredMethod("getInt$");
        int fdint = (Integer) getInt.invoke(fd);       // 反射获取 int fd
        boolean result = mOpenVPNService.protect(fdint); // VpnService.protect()
        if (!result)
            VpnStatus.logWarning("Could not protect VPN socket");
        fdClose(fd);
    } catch (Exception e) {
        VpnStatus.logException("protectFileDescriptor", e);
    }
}
```

##### 场景二：OPENTUN（Java → 原生，传递 TUN 设备 FD）

OpenVPN 需要 TUN 设备进行数据传输，但只有 Java 层能调用 `VpnService.Builder.establish()` 创建 TUN。创建后需要将 FD 传回原生进程。

**流程：**

```
1. 原生 OpenVPN 请求 TUN 设备
2. 调用 management_android_control("OPENTUN", dev) (tun.c:2025)
3. 发送 ">NEED-OK:OPENTUN:MSG:tun" 到 Java 端
4. Java 端处理 NEED-OK:OPENTUN (OpenVpnManagementThread.java:593-597)
5. 调用 OpenVPNService.openTun()
   → VpnService.Builder.establish() → ParcelFileDescriptor (TUN FD)
6. 调用 sendCommandWithFd("needok 'OPENTUN' ok\n", pfd)
   (OpenVpnManagementThread.java:619-651)
7. 通过 LocalSocket.setFileDescriptorsForSend(fd) 附加 FD
8. 写入命令文本 "needok 'OPENTUN' ok\n"
   → FD 作为 ancillary data (SCM_RIGHTS) 一起发送
9. 清除 FD 附件
10. 原生端 man_recv_with_fd() (manage.c:2317-2364)
    → recvmsg() 接收文本 + SCM_RIGHTS FD
11. 存储到 management->connection.lastfdreceived (manage.c:2434)
12. tun.c:2028 取出 tt->fd = management->connection.lastfdreceived
```

**Java 端发送 FD（`OpenVpnManagementThread.java:619-651`）：**

```java
private void sendCommandWithFd(String cmd, ParcelFileDescriptor pfd) {
    FileDescriptor fd = new FileDescriptor();
    // 通过反射设置 FD 的 int 值
    getIntFdMethod.invoke(fd, pfd.getFd());
    FileDescriptor[] fds = {fd};
    mSocket.setFileDescriptorsForSend(fds);
    managmentCommand(cmd);        // 写入命令文本
    mSocket.setFileDescriptorsForSend(null);  // 清除 FD 附件
}
```

**原生端接收 FD（`manage.c:2317-2364`）：**

```c
static int man_recv_with_fd(struct management *man, char *buf, int len) {
    struct msghdr msghdr;
    struct iovec iov;
    char cmsgbuf[CMSG_SPACE(sizeof(int))];
    
    iov.iov_base = buf;
    iov.iov_len = len;
    
    msghdr.msg_control = cmsgbuf;
    msghdr.msg_controllen = sizeof(cmsgbuf);
    
    int ret = recvmsg(man->connection.sd, &msghdr, 0);
    
    // 解析 ancillary data 获取 FD
    for (struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msghdr); cmsg; 
         cmsg = CMSG_NXTHDR(&msghdr, cmsg)) {
        if (cmsg->cmsg_level == SOL_SOCKET && cmsg->cmsg_type == SCM_RIGHTS) {
            man->connection.lastfdreceived = *(int *)CMSG_DATA(cmsg);
        }
    }
    return ret;
}
```

#### 2.4.5 关键源文件索引

**Java/Kotlin 文件：**

| 文件 | 关键行 | 职责 |
|------|--------|------|
| `core/OpenVpnManagementThread.java` | 117-150 | UDS 服务端创建与绑定 |
| `core/OpenVpnManagementThread.java` | 156-167 | 命令发送 (`managmentCommand`) |
| `core/OpenVpnManagementThread.java` | 170-225 | 主读取循环 + FD 接收 |
| `core/OpenVpnManagementThread.java` | 258-329 | 命令分派处理 (`processCommand`) |
| `core/OpenVpnManagementThread.java` | 530-616 | NEED-OK 处理（OPENTUN, IFCONFIG, ROUTE 等） |
| `core/OpenVpnManagementThread.java` | 619-663 | FD 发送（`sendCommandWithFd` + OPENTUN 处理） |
| `core/OpenVpnManagementThread.java` | 666-721 | 密码处理 |
| `core/OpenVpnManagementThread.java` | 729-751 | 网络变化 + 信号发送 |
| `VpnProfile.java` | 407-415 | 生成 `management ... unix` 配置指令 |
| `core/OpenVPNService.java` | 690-741 | 启动管理线程 + 原生进程 |
| `core/OpenVPNService.java` | 852-860 | `openTun()` 创建 TUN 设备 |
| `core/OpenVPNManagement.java` | 1-51 | 管理接口抽象定义 |

**原生 C 文件：**

| 文件 | 关键行 | 职责 |
|------|--------|------|
| `openvpn/manage.c` | 2025-2078 | `man_listen()` — UDS 服务端模式（Android 不使用） |
| `openvpn/manage.c` | 2101-2164 | `man_connect()` — UDS 客户端连接 |
| `openvpn/manage.c` | 2283-2315 | `man_send_with_fd()` — sendmsg + SCM_RIGHTS 发送 FD |
| `openvpn/manage.c` | 2317-2364 | `man_recv_with_fd()` — recvmsg + SCM_RIGHTS 接收 FD |
| `openvpn/manage.c` | 2370-2383 | `management_android_control()` — Android 控制 NEED-OK |
| `openvpn/manage.c` | 2391-2414 | `managment_android_persisttun_action()` — TUN 持久化询问 |
| `openvpn/manage.c` | 2421-2512 | `man_read()` — 读取 + FD 接收 |
| `openvpn/manage.c` | 2514-2557 | `man_write()` — 写入 + FD 发送 |
| `openvpn/manage.c` | 2631-2667 | `man_settings_init()` — UDS 地址初始化 |
| `openvpn/manage.c` | 2731-2773 | `man_connection_init()` — 连接初始化 |
| `openvpn/manage.h` | 35 | `MF_UNIX_SOCK` 标志定义 |
| `openvpn/manage.h` | 244 | `sockaddr_un local_unix` 地址结构 |
| `openvpn/manage.h` | 326-329 | `fdtosend` / `lastfdreceived` FD 传递字段 |
| `openvpn/socket.c` | 3009-3022 | `create_socket_unix()` — `socket(PF_UNIX, ...)` |
| `openvpn/socket.c` | 3059-3066 | `socket_connect_unix()` — `connect()` |
| `openvpn/socket.c` | 3070-3073 | `sockaddr_unix_init()` — 初始化 `sockaddr_un` |
| `openvpn/socket.c` | 3077-3083 | `socket_delete_unix()` — `unlink()` 删除 socket 文件 |
| `openvpn/socket.c` | 731-752 | `protect_fd_nonlocal()` — Android PROTECTFD 流程 |
| `openvpn/tun.c` | 1979-2046 | Android TUN 设备建立（IFCONFIG/ROUTE/OPENTUN） |
| `openvpn/options.c` | 5706-5725 | 解析 `--management <addr> unix` 配置 |
| `openvpn/route.c` | 1539-1550, 1894-1899 | 通过管理接口设置路由 |
| `openvpn/doc/android.txt` | 1-101 | Android 管理协议文档 |

#### 2.4.6 management_io 触发机制

`management_io()` 有两种触发路径：**主事件循环路径**（由 socket I/O 事件驱动）和 **standalone 路径**（由 buffer 写入驱动，脱离主事件循环独立运行）。

##### 路径一：主事件循环路径（socket I/O 事件驱动）

这是最常见的路径，management socket 的 I/O 事件与 tun/socket 事件在同一个事件循环中统一调度。

**完整调用链（点对点模式）：**

```
openvpn_main()                                    openvpn.c:153
  → tunnel_point_to_point()                       openvpn.c:57
    → while (true) {                              openvpn.c:73   ← 主循环
        pre_select(c)
        io_wait(c, p2p_iow_flags(c))              forward.c:2164
          ├── management_socket_set()              注册 management socket
          │     → event_ctl(sd, EVENT_READ/WRITE)
          ├── event_wait()                         阻塞等待 I/O 事件
          │     → 返回就绪事件 → 设置 MANAGEMENT_READ/WRITE 标志位
        process_io(c, c->c2.link_sockets[0])       forward.c:2287
          ├── if (status & MANAGEMENT_READ/WRITE)
          └── management_io(management)            manage.c:3369
                ├── MS_LISTEN       → man_accept()
                ├── MS_CC_WAIT_READ → man_read()
                └── MS_CC_WAIT_WRITE→ man_write()
      }
```

**完整调用链（点对多点/服务器模式）：**

```
openvpn_main()
  → tunnel_server_udp()
    → multi_tcp_process_io() / multi_process_io_udp()
      → while (true) {
          multi_io_wait(multi)                     multi_io.c:158
            ├── management_socket_set()             注册 management socket
            ├── event_wait()                        阻塞等待 I/O 事件
          multi_io_process_io(multi)                multi_io.c:410
            ├── if (e->arg == MULTI_IO_MANAGEMENT)
            └── management_io(management)           manage.c:3369
          }
```

**逐步详解：**

**第一步：注册监听 — `management_socket_set()`**

在 `io_wait()`（`forward.c:2198`）中，调用 `management_socket_set()` 将 management socket 注册到 `event_set` 中，根据当前状态注册不同事件：

| 状态 | 注册事件 | 含义 |
|------|----------|------|
| `MS_LISTEN` | `EVENT_READ` | 监听新管理客户端连接接入 |
| `MS_CC_WAIT_READ` | `EVENT_READ` | 等待管理客户端发来的数据 |
| `MS_CC_WAIT_WRITE` | `EVENT_WRITE` | 等待向管理客户端发送数据 |

源码（`manage.c:3334-3366`，非 Windows 路径）：

```c
void
management_socket_set(struct management *man, struct event_set *es, void *arg,
                      unsigned int *persistent)
{
    switch (man->connection.state)
    {
        case MS_LISTEN:
            if (man_persist_state(persistent, 1))
                event_ctl(es, man->connection.sd_top, EVENT_READ, arg);
            break;
        case MS_CC_WAIT_READ:
            if (man_persist_state(persistent, 2))
                event_ctl(es, man->connection.sd_cli, EVENT_READ, arg);
            break;
        case MS_CC_WAIT_WRITE:
            if (man_persist_state(persistent, 3))
                event_ctl(es, man->connection.sd_cli, EVENT_WRITE, arg);
            break;
    }
}
```

**第二步：事件等待 — `event_wait()`**

`io_wait()` 调用 `event_wait()`（`forward.c:2230`）阻塞等待 I/O 事件。当 management socket 上有活动（可读/可写）时，`event_wait` 返回，返回值中的 `rwflags` 和 `arg`（即 `management_shift=6`）被合成为位掩码写入 `c->c2.event_set_status`：

```c
c->c2.event_set_status |= ((e->rwflags & 3) << shift);
```

- management socket 可读 → 设置 `MANAGEMENT_READ = (1 << (6+0)) = 0x40`
- management socket 可写 → 设置 `MANAGEMENT_WRITE = (1 << (6+1)) = 0x80`

事件位定义（`event.h:59-70`）：

```c
#define SOCKET_SHIFT     0
#define TUN_SHIFT        2
#define ERR_SHIFT        4
#define MANAGEMENT_SHIFT 6
#define MANAGEMENT_READ  (1 << (MANAGEMENT_SHIFT + READ_SHIFT))
#define MANAGEMENT_WRITE (1 << (MANAGEMENT_SHIFT + WRITE_SHIFT))
```

**第三步：分发调用 — `process_io()` / `multi_io_process_io()`**

点对点模式（`forward.c:2292`）：

```c
void process_io(struct context *c, struct link_socket *sock)
{
    const unsigned int status = c->c2.event_set_status;
#ifdef ENABLE_MANAGEMENT
    if (status & (MANAGEMENT_READ | MANAGEMENT_WRITE))
    {
        ASSERT(management);
        management_io(management);
    }
#endif
    // ... 处理其他 I/O 事件（socket/tun/error）
}
```

点对多点模式（`multi_io.c:476`）：

```c
if (e->arg == MULTI_IO_MANAGEMENT)
{
    ASSERT(management);
    management_io(management);
}
```

**第四步：`management_io()` 内部处理**

进入后根据状态分发（`manage.c:3369`，非 Windows 路径）：

```c
void management_io(struct management *man)
{
    switch (man->connection.state)
    {
        case MS_LISTEN:
            man_accept(man);         // 接受新的管理客户端连接
            break;
        case MS_CC_WAIT_READ:
            man_read(man);           // 读取管理客户端命令
            break;
        case MS_CC_WAIT_WRITE:
            man_write(man);          // 向管理客户端发送响应
            break;
        case MS_INITIAL:
            break;
        default:
            ASSERT(0);
    }
}
```

| 状态 | 动作 | 说明 |
|------|------|------|
| `MS_LISTEN` | `man_accept()` | 接受新的管理客户端连接 |
| `MS_CC_WAIT_READ` | `man_read()` | 读取管理客户端发来的命令（如 `needok`、`password` 等） |
| `MS_CC_WAIT_WRITE` | `man_write()` | 向管理客户端发送响应（如 `>NEED-OK:...`、`>STATE:...` 等） |
| `MS_INITIAL` | 无操作 | 管理接口尚未初始化 |

##### 路径二：standalone 路径（buffer 写入驱动，脱离主事件循环）

当 OpenVPN 需要从管理接口**同步等待响应**时（如等待密码输入、等待 NEED-OK 确认、等待 HOLD 释放），会进入 standalone 模式。此时 `management_io()` **不经过主事件循环**，而是由 buffer 写入直接触发，在独立的 mini 事件循环中运行。

**核心机制：`man_output_list_push_finalize()` → `man_output_standalone()` → `management_io()`**

当内部代码调用 `msg(M_CLIENT, ...)` 或 `management_android_control()` 等函数向管理客户端发送消息时，数据先写入 `man->connection.out` 缓冲区，然后调用 `man_output_list_push_finalize()`，该函数会：

1. **`man_update_io_state()`**（`manage.c:254`）：检查输出缓冲区是否有数据，若有则将状态切换到 `MS_CC_WAIT_WRITE`，否则切换到 `MS_CC_WAIT_READ`
2. 若 `standalone_disabled == false`（即处于 standalone 模式），调用 **`man_output_standalone()`**

```c
// manage.c:269-281
static void
man_output_list_push_finalize(struct management *man)
{
    if (management_connected(man))
    {
        man_update_io_state(man);              // 根据缓冲区状态切换 MS_CC_WAIT_*
        if (!man->persist.standalone_disabled)
        {
            volatile int signal_received = 0;
            man_output_standalone(man, &signal_received);  // 直接调用！
        }
    }
}
```

```c
// manage.c:254-267
static void
man_update_io_state(struct management *man)
{
    if (socket_defined(man->connection.sd_cli))
    {
        if (buffer_list_defined(man->connection.out))
            man->connection.state = MS_CC_WAIT_WRITE;   // 缓冲区有数据 → 可写
        else
            man->connection.state = MS_CC_WAIT_READ;    // 缓冲区空 → 可读
    }
}
```

`man_output_standalone()`（`manage.c:3478`）运行一个独立的 mini 事件循环，**立即把缓冲区中的数据发出去**：

```c
// manage.c:3478-3495
static void
man_output_standalone(struct management *man, volatile int *signal_received)
{
    if (man_standalone_ok(man))
    {
        while (man->connection.state == MS_CC_WAIT_WRITE)
        {
            management_io(man);                // 直接调用 management_io()！
            if (man->connection.state == MS_CC_WAIT_WRITE)
            {
                man_block(man, signal_received, 0);  // 短暂阻塞等待 socket 可写
            }
            if (signal_received && *signal_received)
                break;
        }
    }
}
```

**关键区别：** 这里的 `management_io()` 不是由主事件循环的 `event_wait` 触发的，而是因为缓冲区中有了待发送数据（`man->connection.out` 非空），`man_update_io_state()` 将状态设为 `MS_CC_WAIT_WRITE`，`man_output_standalone()` 循环调用 `management_io()` → `man_write()` 将缓冲区数据通过 socket 发送出去，直到缓冲区清空（状态回到 `MS_CC_WAIT_READ`）。

**standalone 模式的典型调用场景：**

以下场景中，OpenVPN 会进入 standalone 事件循环，等待管理客户端的响应：

| 场景 | 入口函数 | 调用链 |
|------|----------|--------|
| 等待用户名/密码 | `management_query_user_pass()` (manage.c:3610) | `msg(M_CLIENT, ">PASSWORD:...")` → `man_output_list_push_finalize()` → `man_output_standalone()` → `management_io()`；然后进入 `man_standalone_event_loop()` 循环等待客户端回复 |
| 等待 NEED-OK 确认（如 PROTECTFD/OPENTUN） | `management_android_control()` (manage.c:2370) | 同上 |
| 等待 HOLD 释放 | `management_hold()` | `msg(M_CLIENT, ">HOLD:...")` → 同上 |
| 等待 NEED-STR 输入 | `management_query_multiline()` (manage.c:3741) | 同上 |
| 定时事件循环 | `management_event_loop_n_seconds()` (manage.c:3557) | 直接调用 `man_standalone_event_loop()` → `man_block()` + `management_io()` |

这些场景的共同特点是：**OpenVPN 主流程被阻塞，必须等管理客户端回复才能继续**。此时主事件循环（`io_wait` / `process_io`）暂停，standalone 模式接管 management 接口的 I/O 处理。

**`standalone_disabled` 标志位控制：**

```c
// manage.c:2870 — 注册回调时禁用 standalone
void management_set_callback(struct management *man, const struct management_callback *cb)
{
    man->persist.standalone_disabled = true;   // 主事件循环接管时禁用 standalone
    man->persist.callback = *cb;
}

// manage.c:2877 — 清除回调时恢复 standalone
void management_clear_callback(struct management *man)
{
    man->persist.standalone_disabled = false;  // 允许 standalone 模式
    ...
}
```

- **`standalone_disabled = true`**（默认运行态）：`man_output_list_push_finalize()` 不调用 `man_output_standalone()`，数据留在缓冲区，等主事件循环的下一次 `io_wait` → `event_wait` → `process_io` → `management_io()` 来发送
- **`standalone_disabled = false`**（同步等待场景）：`man_output_list_push_finalize()` 立即调用 `man_output_standalone()`，脱离主事件循环立即发送缓冲区数据

但在同步等待场景中，即使 `standalone_disabled` 被临时设为 `false`，进入 `management_query_user_pass()` 等函数时也会先保存原始值、设为 `false`，结束后恢复（`manage.c:3619-3627`、`manage.c:3749-3755`），确保 standalone 模式在这些函数中始终生效。

**standalone 模式的独立事件循环：**

`man_standalone_event_loop()`（`manage.c:3500`）也有自己的 `event_wait`，但它只关注 management socket：

```c
// manage.c:3500-3514
static int
man_standalone_event_loop(struct management *man, volatile int *signal_received,
                          const time_t expire)
{
    int status = -1;
    if (man_standalone_ok(man))
    {
        status = man_block(man, signal_received, expire);  // 只监听 management socket
        if (status > 0)
        {
            management_io(man);   // 有事件时处理
        }
    }
    return status;
}
```

```c
// manage.c:3418-3444
static int
man_block(struct management *man, volatile int *signal_received, const time_t expire)
{
    // ... 只注册 management socket 到 event_set
    event_reset(man->connection.es);
    management_socket_set(man, man->connection.es, NULL, NULL);
    status = event_wait(man->connection.es, &tv, &esr, 1);  // 只等待 management socket
    // ...
}
```

##### 两条路径对比

| 对比维度 | 路径一：主事件循环 | 路径二：standalone |
|----------|-------------------|-------------------|
| 触发源 | socket I/O 事件（`event_wait` 返回） | buffer 写入（`man_output_list_push_finalize`） |
| 事件循环 | 主循环 `io_wait` + `process_io` | 独立 mini 循环 `man_output_standalone` / `man_standalone_event_loop` |
| 监听的 fd | tun + socket + management（全部） | 只监听 management socket |
| 调用上下文 | `tunnel_point_to_point` 主循环 | `management_query_user_pass` 等同步等待函数 |
| `standalone_disabled` | `true`（不触发 standalone） | `false`（临时关闭，允许 standalone） |
| 典型场景 | 正常数据收发（状态通知、日志推送） | 等待密码、NEED-OK 确认、HOLD 释放 |
| 数据发送时机 | 下一次主循环迭代时 | 立即发送（循环调用 `management_io()` 直到缓冲区清空） |

##### 完整触发链路总图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        路径一：主事件循环                                │
│                                                                         │
│  tunnel_point_to_point()                    openvpn.c:57                │
│    while (true) {                                                       │
│      pre_select(c)                                                      │
│      io_wait(c, flags)                      forward.c:2164             │
│        ├── management_socket_set()          注册 management socket      │
│        ├── event_wait()                     阻塞等待所有 I/O            │
│        │     → MANAGEMENT_READ/WRITE 标志位设置                          │
│      process_io(c, sock)                    forward.c:2287             │
│        └── if (status & MANAGEMENT_READ/WRITE)                          │
│            management_io(management)        manage.c:3369              │
│    }                                                                    │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                        路径二：standalone（buffer 驱动）                  │
│                                                                         │
│  management_query_user_pass()               manage.c:3610  (等密码)     │
│  management_android_control()               manage.c:2370  (PROTECTFD)  │
│  management_hold()                          (等HOLD释放)                │
│  management_query_multiline()               manage.c:3741  (等输入)     │
│    │                                                                    │
│    ├── msg(M_CLIENT, ">NEED-OK:...")                                    │
│    │     → virtual_output_callback_func()  manage.c:360                │
│    │       → man_output_list_push_str()    manage.c:284  ← 写入缓冲区  │
│    │                                                                    │
│    ├── man_output_list_push_finalize()      manage.c:270                │
│    │     ├── man_update_io_state()          manage.c:254                │
│    │     │     → 缓冲区有数据 → state = MS_CC_WAIT_WRITE               │
│    │     └── man_output_standalone()        manage.c:3478               │
│    │           while (state == MS_CC_WAIT_WRITE) {                      │
│    │             management_io(management)  manage.c:3369  ← 立即发送！ │
│    │               └── man_write()          manage.c:2514               │
│    │             man_block()                只等 management socket      │
│    │           }                                                        │
│    │                                                                    │
│    └── man_standalone_event_loop()          manage.c:3500  ← 等客户端回复│
│          ├── man_block()                    只监听 management socket     │
│          │     → management_socket_set()                                │
│          │     → event_wait()                                           │
│          └── management_io(management)      有事件时处理                 │
│                └── man_read()               读取客户端回复              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.5 前台服务与生命周期管理

OpenVPNService 作为 Android 前台服务运行，具有以下特性：

1. **前台通知**：连接期间始终显示通知，包含连接状态、流量统计、暂停/断开操作
2. **`START_STICKY`**：系统杀死服务后自动重启（配合 `keepVPNAlive` JobService）
3. **Always-on VPN 支持**：Android 的 Always-on VPN 功能会在连接断开时自动重启
4. **`onRevoke()` 处理**：用户撤销 VPN 权限时，发送 `SIGINT` 给 OpenVPN 进程并清理

---

## 3. 项目目录结构

```
ics-openvpn/
├── main/                                # 主应用模块
│   └── src/
│       ├── main/                        # 核心源码集（:openvpn 进程）
│       │   ├── java/de/blinkt/openvpn/
│       │   │   ├── LaunchVPN.java           # VPN 启动 Activity
│       │   │   ├── VpnProfile.java          # VPN 配置模型
│       │   │   ├── OnBootReceiver.java      # 开机启动接收器
│       │   │   ├── NotifyHelper.java        # 通知辅助类
│       │   │   ├── FileProvider.java         # 文件提供者
│       │   │   ├── api/                     # 外部 API 层
│       │   │   │   ├── ExternalOpenVPNService.java  # 外部 API AIDL 实现
│       │   │   │   ├── RemoteAction.java           # 外部 Intent 处理
│       │   │   │   ├── ConfirmDialog.java          # 外部 App 权限确认
│       │   │   │   ├── GrantPermissionsActivity.java # VPN 权限授予
│       │   │   │   ├── AppRestrictions.java        # MDM 托管配置
│       │   │   │   ├── ExternalAppDatabase.java    # 允许的外部 App 数据库
│       │   │   │   ├── APIVpnProfile.java          # AIDL 配置数据类
│       │   │   │   └── SecurityRemoteException.java
│       │   │   ├── activities/
│       │   │   │   └── DisconnectVPN.java    # 断开/重连对话框
│       │   │   └── core/                    # 核心引擎层
│       │   │       ├── OpenVPNService.java          # 核心 VpnService
│       │   │       ├── OpenVPNStatusService.java    # 状态桥接 Service
│       │   │       ├── OpenVPNManagement.java       # 管理接口定义
│       │   │       ├── OpenVpnManagementThread.java # 管理接口实现
│       │   │       ├── OpenVPNThread.java           # OpenVPN 2.x 进程线程
│       │   │       ├── VPNLaunchHelper.java         # 启动辅助
│       │   │       ├── VpnStatus.java               # 全局状态中心
│       │   │       ├── ConnectionStatus.java        # 连接状态枚举
│       │   │       ├── Connection.java              # 连接端点模型
│       │   │       ├── ProfileManager.java          # 配置管理器
│       │   │       ├── ConfigParser.java            # .ovpn 配置解析器
│       │   │       ├── DeviceStateReceiver.java     # 设备状态监听
│       │   │       ├── keepVPNAlive.java            # 保活 JobService
│       │   │       ├── NativeUtils.java             # JNI 工具
│       │   │       ├── NetworkSpace.java            # IP 路由计算
│       │   │       ├── NetworkUtils.java            # 网络工具
│       │   │       ├── CIDRIP.java                  # CIDR 表示
│       │   │       ├── LogItem.java                 # 日志条目
│       │   │       ├── TrafficHistory.java          # 流量历史
│       │   │       ├── PasswordCache.java           # 密码内存缓存
│       │   │       ├── Preferences.java             # 偏好设置
│       │   │       ├── GlobalPreferences.java       # 全局偏好
│       │   │       ├── OrbotHelper.java             # Tor/Orbot 集成
│       │   │       ├── ProxyDetection.java          # 代理检测
│       │   │       ├── ExtAuthHelper.java           # 外部认证辅助
│       │   │       ├── X509Utils.java               # X509 证书工具
│       │   │       ├── ICSOpenVPNApplication.java   # Application 类
│       │   │       ├── StatusListener.java          # UI 进程状态监听
│       │   │       ├── LocaleHelper.java            # 语言处理
│       │   │       └── LogFileHandler.java          # 日志文件写入
│       │   ├── aidl/                        # AIDL 定义
│       │   │   └── de/blinkt/openvpn/
│       │   │       ├── api/
│       │   │       │   ├── IOpenVPNAPIService.aidl
│       │   │       │   ├── IOpenVPNStatusCallback.aidl
│       │   │       │   ├── APIVpnProfile.aidl
│       │   │       │   └── ExternalCertificateProvider.aidl
│       │   │       └── core/
│       │   │           ├── IOpenVPNServiceInternal.aidl
│       │   │           ├── IServiceStatus.aidl
│       │   │           ├── IStatusCallbacks.aidl
│       │   │           ├── ConnectionStatus.aidl
│       │   │           ├── LogItem.aidl
│       │   │           └── TrafficHistory.aidl
│       │   ├── cpp/                          # 原生 C/C++ 代码
│       │   │   ├── openvpn/                  # OpenVPN 源码
│       │   │   ├── openssl/                  # OpenSSL
│       │   │   ├── mbedtls/                  # mbedTLS
│       │   │   └── lzo/                      # LZO 压缩
│       │   └── AndroidManifest.xml
│       ├── ui/                              # UI 源码集（默认进程）
│       │   └── java/de/blinkt/openvpn/
│       │       ├── activities/              # MainActivity, LogWindow, VPNPreferences, ConfigConverter 等
│       │       ├── fragments/               # Settings, Log, Graph, Faq, ViewLog 等片段
│       │       └── views/                   # 自定义 UI 组件
│       ├── skeleton/                        # 精简 UI 变体
│       ├── test/                            # 单元测试
│       └── testui/                          # UI 测试
├── remoteExample/                          # 外部 API 示例 App
├── tlsexternalcertprovider/                # 外部证书提供者示例
└── doc/                                    # 文档
```

---

## 4. 核心类详解

### 4.1 VpnStatus — 全局状态中心

**文件**: `core/VpnStatus.java`

VpnStatus 是整个项目的状态枢纽，采用**单例 + 观察者**模式。每个进程有自己独立的 VpnStatus 实例。

**核心职责**：
- 维护当前连接状态（`mLastLevel`, `mLaststate`, `mLaststatemsg`）
- 管理日志缓冲区（`logbuffer`，最多 1000 条）
- 管理流量历史（`trafficHistory`）
- 管理四个观察者列表并分发事件：
  - `LogListener` → 日志事件
  - `StateListener` → 连接状态事件
  - `ByteCountListener` → 流量统计事件
  - `ProfileNotifyListener` → 配置版本变更事件

**关键方法**：

| 方法 | 作用 |
|------|------|
| `updateStateString(state, msg, resid, level, intent)` | 更新连接状态，通知所有 StateListener |
| `newLogItem(logItem)` | 添加日志条目，通知所有 LogListener |
| `updateByteCount(in, out)` | 更新流量统计，通知所有 ByteCountListener |
| `setConnectedVPNProfile(uuid)` | 设置已连接的 VPN UUID，通知所有 StateListener |
| `notifyProfileVersionChanged(uuid, version, changedInThisProcess)` | 通知配置版本变更 |

**状态映射**（`VpnStatus.java:334-357`）：

| OpenVPN 状态字符串 | ConnectionStatus 级别 |
|---|---|
| CONNECTING, WAIT, RECONNECTING, RESOLVE, TCP_CONNECT | `LEVEL_CONNECTING_NO_SERVER_REPLY_YET` |
| AUTH, GET_CONFIG, ASSIGN_IP, ADD_ROUTES, AUTH_PENDING | `LEVEL_CONNECTING_SERVER_REPLIED` |
| CONNECTED | `LEVEL_CONNECTED` |
| DISCONNECTED, EXITING | `LEVEL_NOTCONNECTED` |

### 4.2 OpenVPNService — VPN 核心服务

**文件**: `core/OpenVPNService.java`

继承自 `android.net.VpnService`，运行在 `:openvpn` 进程，是整个 VPN 引擎的核心。

**核心职责**：
- 管理原生 OpenVPN 进程的生命周期
- 创建和配置 TUN 设备（`VpnService.Builder`）
- 前台通知管理
- 实现 `IOpenVPNServiceInternal` AIDL 接口供外部控制
- 监听 `VpnStatus` 状态更新通知

**关键方法**：

| 方法 | 作用 |
|------|------|
| `onStartCommand(intent, flags, startId)` | 接收启动 Intent，处理 PAUSE/RESUME/START |
| `startOpenVPN(intent, startId)` | 启动 VPN：获取配置、创建管理线程、启动原生进程 |
| `openTun()` | 配置并建立 TUN 设备，返回 ParcelFileDescriptor |
| `onBind(intent)` | 返回 `IOpenVPNServiceInternal.Stub` binder |
| `stopVPN(replaceConnection)` | 通过管理接口发送 SIGINT 停止 OpenVPN |
| `userPause(shouldBePaused)` | 暂停/恢复 VPN |
| `protect(fd)` | 保护 socket 不经过 VPN（防止回环） |
| `endVpnService()` | 清理资源：移除监听器、注销 Receiver、停止服务 |
| `updateState(...)` | StateListener 回调：更新通知栏、广播状态 |

**启动流程核心代码**（`OpenVPNService.java:651-716`）：
```java
private void startOpenVPN(Intent intent, int startId) {
    VpnProfile vp = fetchVPNProfile(intent);
    if (vp == null) { stopSelf(startId); return; }
    if (!checkVPNPermission(vp)) return;
    
    mProfile = vp;
    ProfileManager.setConnectedVpnProfile(this, vp);
    VpnStatus.setConnectedVPNProfile(vp.getUUIDString());
    keepVPNAlive.scheduleKeepVPNAliveJobService(this, vp);
    
    String[] argv = VPNLaunchHelper.buildOpenvpnArgv(this);
    stopOldOpenVPNProcess(mManagement, mOpenVPNThread);
    
    if (!useOpenVPN3) {
        OpenVpnManagementThread ovpnManagementThread = new OpenVpnManagementThread(mProfile, this);
        if (ovpnManagementThread.openManagementInterface(this)) {
            Thread mSocketManagerThread = new Thread(ovpnManagementThread, "OpenVPNManagementThread");
            mSocketManagerThread.start();
            mManagement = ovpnManagementThread;
        }
    }
    // ... 启动原生进程线程
}
```

**IOpenVPNServiceInternal.Stub 实现**（`OpenVPNService.java:123-157`）：
```java
private final IBinder mBinder = new IOpenVPNServiceInternal.Stub() {
    @Override public boolean protect(int fd) { return OpenVPNService.this.protect(fd); }
    @Override public void userPause(boolean b) { OpenVPNService.this.userPause(b); }
    @Override public boolean stopVPN(boolean replaceConnection) { return OpenVPNService.this.stopVPN(replaceConnection); }
    @Override public void addAllowedExternalApp(String packagename) { ... }
    @Override public boolean isAllowedExternalApp(String packagename) { ... }
    @Override public void challengeResponse(String response) { ... }
};
```

### 4.3 OpenVPNStatusService — 状态桥接服务

**文件**: `core/OpenVPNStatusService.java`

运行在 `:openvpn` 进程，是状态从 VPN 引擎传递到 UI 进程的桥梁。

**核心职责**：
- 监听 VpnStatus 的所有事件（日志、状态、流量、配置变更）
- 通过 `IServiceStatus` AIDL 接口向 UI 进程注册回调
- 通过 `IStatusCallbacks` 将事件转发给已注册的 UI 客户端
- 注册回调时返回历史日志的 Pipe（`ParcelFileDescriptor`）

**Handler 分发机制**（`OpenVPNStatusService.java:186-241`）：

所有 VpnStatus 事件通过 `OpenVPNStatusHandler`（静态 Handler）分发，避免阻塞发送线程：

```java
private static class OpenVPNStatusHandler extends Handler {
    WeakReference<OpenVPNStatusService> service = null;
    
    @Override
    public void handleMessage(Message msg) {
        RemoteCallbackList<IStatusCallbacks> callbacks = service.get().mCallbacks;
        final int N = callbacks.beginBroadcast();
        for (int i = 0; i < N; i++) {
            IStatusCallbacks broadcastItem = callbacks.getBroadcastItem(i);
            switch (msg.what) {
                case SEND_NEW_LOGITEM:    broadcastItem.newLogItem((LogItem) msg.obj); break;
                case SEND_NEW_BYTECOUNT:  broadcastItem.updateByteCount(inout.first, inout.second); break;
                case SEND_NEW_STATE:      sendUpdate(broadcastItem, (UpdateMessage) msg.obj); break;
                case SEND_NEW_CONNECTED_VPN: broadcastItem.connectedVPN((String) msg.obj); break;
                case SEND_NEW_PROFILE_VERSION: broadcastItem.notifyProfileVersionChanged(...); break;
            }
        }
        callbacks.finishBroadcast();
    }
}
```

**历史日志传递**（`OpenVPNStatusService.java:63-105`）：

当 UI 进程注册回调时，`registerStatusCallback()` 通过 `ParcelFileDescriptor.createPipe()` 创建管道，将日志缓冲区序列化后写入管道，UI 进程在独立线程中读取管道恢复历史日志。

### 4.4 OpenVpnManagementThread — 管理接口线程

**文件**: `core/OpenVpnManagementThread.java`

实现了 `OpenVPNManagement` 接口和 `Runnable`，负责与原生 OpenVPN 进程通过 Unix Domain Socket 通信。

**核心流程**：

1. **创建服务端 Socket**：`openManagementInterface()` 在 `cacheDir/mgmtsocket` 创建 LocalServerSocket
2. **等待连接**：`run()` 中调用 `mServerSocket.accept()` 等待 OpenVPN 进程连接
3. **发送初始命令**：`version 3`, `hold release`, `bytecount 2`, `state on`
4. **循环读取**：持续读取 OpenVPN 发来的命令并分派处理

**命令处理**（`processCommand()` 方法，`OpenVpnManagementThread.java:273-329`）：

| OpenVPN 命令 | 处理方式 |
|---|---|
| `>PASSWORD:` | 提供用户名/密码/私钥密码（从 PasswordCache 获取） |
| `>NEED-OK:OPENTUN` | 调用 `OpenVPNService.openTun()` 创建 TUN，通过 ancillary data 传递 FD |
| `>NEED-OK:DNSSERVER` | 记录 DNS 服务器到 tunConfig |
| `>NEED-OK:ROUTE` | 记录路由到 tunConfig |
| `>NEED-OK:IFCONFIG` | 设置本地 IP 和 MTU |
| `>STATE:` | 解析状态并调用 `VpnStatus.updateStateString()` |
| `>BYTECOUNT:` | 调用 `VpnStatus.updateByteCount()` |
| `>LOG:` | 调用 `VpnStatus.logMessageOpenVPN()` |
| `>PROXY:` | 处理代理设置 |
| `>PK_SIGN:` | 处理 PKCS#11 或外部 App 签名 |
| `>HOLD:` | 处理重连等待 |
| `>INFOMSG:OPEN_URL/CR_TEXT/WEB_AUTH` | 处理 SSO 认证 |
| `PROTECTFD:` | 调用 `VpnService.protect()` 保护 socket |

### 4.5 VPNLaunchHelper — 启动辅助类

**文件**: `core/VPNLaunchHelper.java`

负责准备原生 OpenVPN 二进制文件和启动 OpenVPNService。

**核心方法**：

| 方法 | 作用 |
|------|------|
| `buildOpenvpnArgv(Context c)` | 构建命令行参数：`[libovpnexec.so路径, --config, stdin]` |
| `writeMiniVPN(Context context)` | 确定 native 二进制文件路径（Android P+ 使用 nativeLibraryDir） |
| `startOpenVpn(profile, context, reason, replace)` | 构建 Intent 并调用 `startForegroundService()` 启动 OpenVPNService |

### 4.6 VpnProfile — 配置模型

**文件**: `VpnProfile.java`

表示一个 VPN 配置文件，包含：
- 连接参数（服务器地址、端口、协议）
- 认证信息（证书、私钥、用户名/密码）
- 配置生成（`getConfigFile()` 生成 OpenVPN 配置文本）
- 配置文件导入（配合 `ConfigParser` 解析 .ovpn 文件）

### 4.7 ProfileManager — 配置管理器

**文件**: `core/ProfileManager.java`

单例，管理所有 VpnProfile 的 CRUD 操作：
- 从文件系统加载/保存配置（加密 `.cp` 或明文 `.vp` 文件）
- 追踪当前连接的配置
- 管理临时配置（外部 App 创建的 inline 配置）

### 4.8 StatusListener — UI 进程状态监听器

**文件**: `core/StatusListener.java`

运行在 **UI 进程**，负责将 `:openvpn` 进程的状态同步到 UI 进程的 VpnStatus 单例。

**初始化**（`StatusListener.java:136-147`）：
```java
void init(Context c) {
    Intent intent = new Intent(c, OpenVPNStatusService.class);
    intent.setAction(OpenVPNService.START_SERVICE);
    c.bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
}
```

**绑定成功后**（`StatusListener.java:79-131`）：
1. 判断是否跨进程（`queryLocalInterface` 返回 null 表示跨进程）
2. 跨进程时：注册 `IStatusCallbacks` 回调，通过管道读取历史日志
3. 同进程时：初始化日志缓存

**回调实现**（`StatusListener.java:36-75`）：
```java
private final IStatusCallbacks mCallback = new IStatusCallbacks.Stub() {
    @Override public void newLogItem(LogItem item) { VpnStatus.newLogItem(item); }
    @Override public void updateStateString(String state, String msg, int resid, 
                                            ConnectionStatus level, Intent intent) {
        VpnStatus.updateStateString(state, msg, resid, level, reCreateIntent(intent));
    }
    @Override public void updateByteCount(long inBytes, long outBytes) { VpnStatus.updateByteCount(inBytes, outBytes); }
    @Override public void connectedVPN(String uuid) { VpnStatus.setConnectedVPNProfile(uuid); }
    @Override public void notifyProfileVersionChanged(String uuid, int version) {
        ProfileManager.notifyProfileVersionChanged(mContext, uuid, version);
    }
};
```

### 4.9 DeviceStateReceiver — 设备状态监听

**文件**: `core/DeviceStateReceiver.java`

监听系统广播，实现 VPN 暂停/恢复逻辑：
- **网络变化**（`CONNECTIVITY_ACTION`）：网络断开时暂停，恢复时重连
- **屏幕关闭/打开**：根据用户设置决定是否暂停 VPN
- **用户暂停**：用户主动暂停/恢复

### 4.10 keepVPNAlive — 保活机制

**文件**: `core/keepVPNAlive.java`

继承 `JobService`，定期检查 VPN 是否在运行，如果断开则自动重启。

- `scheduleKeepVPNAliveJobService()`：使用 JobScheduler 注册周期性任务
- `onStartJob()`：检查当前状态，如果 `VpnStatus.isVPNActive()` 返回 false，则重新启动 VPN

---

## 5. AIDL 接口详解

### 5.1 内部 AIDL 接口

#### IOpenVPNServiceInternal.aidl

```aidl
interface IOpenVPNServiceInternal {
    boolean protect(int fd);              // 保护 socket 不走 VPN
    void userPause(boolean b);            // 用户暂停/恢复
    boolean stopVPN(boolean replaceConnection);  // 停止 VPN
    void addAllowedExternalApp(String packagename);  // 添加允许的外部 App
    boolean isAllowedExternalApp(String packagename); // 检查外部 App 权限
    void challengeResponse(String response);  // CR_TEXT 响应
}
```

**实现者**: `OpenVPNService.mBinder`（`OpenVPNService.java:123`）
**消费者**: `ExternalOpenVPNService`, `DisconnectVPN`, `RemoteAction`, `OpenVPNTileService`

#### IServiceStatus.aidl

```aidl
interface IServiceStatus {
    ParcelFileDescriptor registerStatusCallback(IStatusCallbacks cb);  // 注册回调，返回历史日志管道
    void unregisterStatusCallback(IStatusCallbacks cb);                // 注销回调
    String getLastConnectedVPN();                                      // 获取最近连接的 VPN UUID
    void setCachedPassword(String uuid, int type, String password);    // 设置缓存密码（跨进程传递密码）
    TrafficHistory getTrafficHistory();                                // 获取流量历史
    oneway void notifyProfileVersionChanged(String uuid, int version); // 通知配置版本变更
}
```

**实现者**: `OpenVPNStatusService.mBinder`（`OpenVPNStatusService.java:60`）
**消费者**: `StatusListener`（UI 进程）

#### IStatusCallbacks.aidl

```aidl
oneway interface IStatusCallbacks {
    oneway void newLogItem(LogItem item);
    oneway void updateStateString(String state, String msg, int resid, ConnectionStatus level, Intent intent);
    oneway void updateByteCount(long inBytes, long outBytes);
    oneway void connectedVPN(String uuid);
    oneway void notifyProfileVersionChanged(String uuid, int profileVersion);
}
```

**实现者**: `StatusListener.mCallback`（UI 进程，`StatusListener.java:36`）
**消费者**: `OpenVPNStatusService`（通过 `RemoteCallbackList` 广播）

所有方法标记为 `oneway`，表示异步调用，不阻塞发送方。

### 5.2 外部 API AIDL 接口

#### IOpenVPNAPIService.aidl

```aidl
interface IOpenVPNAPIService {
    List<APIVpnProfile> getProfiles();                    // 获取所有配置
    void startProfile(String profileUUID);                // 通过 UUID 启动配置
    boolean addVPNProfile(String name, String config);    // 添加配置（旧接口）
    void startVPN(String inlineconfig);                   // 使用 inline 配置启动
    Intent prepare(String packagename);                   // 检查/请求外部 App 权限
    Intent prepareVPNService();                           // 检查/请求 VPN 权限
    void disconnect();                                    // 断开 VPN
    void pause();                                         // 暂停 VPN
    void resume();                                        // 恢复 VPN
    void registerStatusCallback(IOpenVPNStatusCallback cb);   // 注册状态回调
    void unregisterStatusCallback(IOpenVPNStatusCallback cb); // 注销状态回调
    void removeProfile(String profileUUID);               // 删除配置
    boolean protectSocket(ParcelFileDescriptor fd);       // 保护 socket
    APIVpnProfile addNewVPNProfile(String name, boolean userEditable, String config); // 添加配置（新接口）
    void startVPNwithExtras(String inlineconfig, Bundle extras); // 带 extras 启动
    APIVpnProfile addNewVPNProfileWithExtras(String name, boolean userEditable, String config, Bundle extras);
    APIVpnProfile getDefaultProfile();                    // 获取默认配置
    void setDefaultProfile(String profileUUID);           // 设置默认配置
}
```

**实现者**: `ExternalOpenVPNService.mBinder`（`ExternalOpenVPNService.java:119`）
**消费者**: 第三方应用（如 `remoteExample/` 中的示例 App）

#### IOpenVPNStatusCallback.aidl

```aidl
oneway interface IOpenVPNStatusCallback {
    oneway void newStatus(String uuid, String state, String message, String level);
}
```

**实现者**: 第三方应用提供的回调
**消费者**: `ExternalOpenVPNService`（通过 `RemoteCallbackList` 广播）

#### APIVpnProfile.aidl

```aidl
parcelable APIVpnProfile;
```

字段：`uuid`, `name`, `userEditable`, `profileCreator`

### 5.3 外部证书提供者 AIDL

#### ExternalCertificateProvider.aidl

```aidl
interface ExternalCertificateProvider {
    byte[] getSignedData(String alias, byte[] data);           // 已废弃，用下面的代替
    byte[] getCertificateChain(String alias);                  // 获取证书链
    Bundle getCertificateMetaData(String alias);               // 获取证书元数据
    byte[] getSignedDataWithExtra(String alias, byte[] data, Bundle extra);  // 签名数据
}
```

**实现者**: 第三方证书提供应用（示例：`tlsexternalcertprovider/ExternalCertService`）
**消费者**: `ExtAuthHelper`（在 `:openvpn` 进程中，绑定外部证书 App 获取签名）

### 5.4 Parcelable 数据类

| 类 | AIDL 声明 | 用途 |
|---|---|---|
| `ConnectionStatus` | `parcelable ConnectionStatus` | VPN 连接状态枚举（Parcelable） |
| `LogItem` | `parcelable LogItem` | 日志条目（包含级别、时间、消息、资源 ID） |
| `TrafficHistory` | `parcelable TrafficHistory` | 流量统计历史记录 |
| `APIVpnProfile` | `parcelable APIVpnProfile` | 外部 API 用的配置摘要 |

---

## 6. Service 详解及与 AIDL 的关系

### 6.1 OpenVPNService

| 属性 | 值 |
|---|---|
| 继承 | `android.net.VpnService` |
| 进程 | `:openvpn` |
| AIDL 接口 | `IOpenVPNServiceInternal` |
| AIDL 实现 | `mBinder`（`IOpenVPNServiceInternal.Stub`） |
| Intent Filter | `android.net.VpnService` |
| 权限 | `android.permission.BIND_VPN_SERVICE` |
| 前台服务类型 | `specialUse` (VPN) |

**onBind 行为**（`OpenVPNService.java:228-234`）：
```java
@Override
public IBinder onBind(Intent intent) {
    String action = intent.getAction();
    if (action != null && action.equals(START_SERVICE))
        return mBinder;  // 返回 IOpenVPNServiceInternal.Stub
    else
        return super.onBind(intent);  // VpnService 的默认 binder（用于 VPN 权限撤销）
}
```

当 action 为 `START_SERVICE` 时，返回自定义 binder 供其他组件控制 VPN；否则返回 VpnService 默认 binder。

**VpnStatus 监听**：
- 注册为 `StateListener`：更新通知栏、发送广播
- 注册为 `ByteCountListener`：更新通知栏流量信息

### 6.2 OpenVPNStatusService

| 属性 | 值 |
|---|---|
| 继承 | `android.app.Service` |
| 进程 | `:openvpn` |
| AIDL 接口 | `IServiceStatus` |
| AIDL 实现 | `mBinder`（`IServiceStatus.Stub`） |
| 回调接口 | `IStatusCallbacks`（通过 `RemoteCallbackList` 管理） |
| exported | `false`（仅供内部使用） |

**事件流向**：
```
VpnStatus (观察者模式)
    ├── newLog(logItem)           → mHandler → SEND_NEW_LOGITEM    → IStatusCallbacks.newLogItem()
    ├── updateState(...)          → mHandler → SEND_NEW_STATE      → IStatusCallbacks.updateStateString()
    ├── updateByteCount(in,out)   → mHandler → SEND_NEW_BYTECOUNT  → IStatusCallbacks.updateByteCount()
    ├── setConnectedVPN(uuid)     → mHandler → SEND_NEW_CONNECTED_VPN → IStatusCallbacks.connectedVPN()
    └── notifyProfileVersionChanged → mHandler → SEND_NEW_PROFILE_VERSION → IStatusCallbacks.notifyProfileVersionChanged()
```

**为什么使用 Handler 而不是直接调用？**

VpnStatus 的监听器回调可能在任意线程执行。使用 Handler 将事件投递到主线程消息队列，然后通过 `RemoteCallbackList` 遍历所有注册的 AIDL 回调。这样可以：
1. 避免阻塞 VpnStatus 的事件分发线程
2. `IStatusCallbacks` 方法标记为 `oneway`，跨进程调用本身就是异步的
3. Handler 使用 `WeakReference` 持有 Service，防止内存泄漏

**历史日志管道机制**：

`registerStatusCallback()` 不仅注册回调，还返回一个 `ParcelFileDescriptor` 管道：
```java
final ParcelFileDescriptor[] pipe = ParcelFileDescriptor.createPipe();
new Thread("pushLogs") {
    public void run() {
        DataOutputStream fd = new DataOutputStream(new ParcelFileDescriptor.AutoCloseOutputStream(pipe[1]));
        // 等待日志文件读取完成
        synchronized (VpnStatus.readFileLock) { ... }
        // 写入所有历史日志
        for (LogItem logItem : logbuffer) {
            byte[] bytes = logItem.getMarschaledBytes();
            fd.writeShort(bytes.length);
            fd.write(bytes);
        }
        fd.writeShort(0x7fff);  // 结束标记
        fd.close();
    }
}.start();
return pipe[0];  // 读取端返回给客户端
```

UI 进程的 `StatusListener` 在独立线程中读取管道，恢复历史日志。

### 6.3 ExternalOpenVPNService

| 属性 | 值 |
|---|---|
| 继承 | `android.app.Service` |
| 进程 | `:openvpn` |
| AIDL 接口 | `IOpenVPNAPIService` |
| AIDL 实现 | `mBinder`（`IOpenVPNAPIService.Stub`） |
| 回调接口 | `IOpenVPNStatusCallback`（通过 `RemoteCallbackList` 管理） |
| exported | `true`（允许外部 App 绑定） |
| Intent Filter | `de.blinkt.openvpn.api.IOpenVPNAPIService` |

**内部绑定关系**：

`ExternalOpenVPNService` 在 `onCreate()` 中绑定 `OpenVPNService`：
```java
Intent intent = new Intent(getBaseContext(), OpenVPNService.class);
intent.setAction(OpenVPNService.START_SERVICE);
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
```

通过 `IOpenVPNServiceInternal` 代理调用 `stopVPN()`, `userPause()`, `protect()` 等方法。

**权限模型**：

1. 外部 App 调用 `prepare(packageName)` → 检查是否在 `ExternalAppDatabase` 白名单
2. 如果不在白名单 → 返回 `ConfirmDialog` Intent，弹出确认对话框
3. 用户确认后 → App 被添加到白名单
4. 后续调用通过 `checkOpenVPNPermission()` 验证

**状态广播**：

`ExternalOpenVPNService` 也实现 `VpnStatus.StateListener`，将状态变化通过 `IOpenVPNStatusCallback` 广播给外部 App：

```
VpnStatus.updateStateString()
    → ExternalOpenVPNService.updateState()
        → mHandler → SEND_TOALL
            → IOpenVPNStatusCallback.newStatus(uuid, state, message, level)
```

### 6.4 keepVPNAlive

| 属性 | 值 |
|---|---|
| 继承 | `android.app.job.JobService` |
| 进程 | `:openvpn` |
| 权限 | `android.permission.BIND_JOB_SERVICE` |

不是 AIDL Service，而是 JobService。通过 JobScheduler 周期性调度，检查 VPN 是否活跃，如果不活跃则自动重启。

### 6.5 Service 与 AIDL 绑定关系总览

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              :openvpn 进程                                      │
│                                                                                 │
│  ┌───────────────────────────┐   ┌───────────────────────────────────────────┐  │
│  │    OpenVPNService         │   │    OpenVPNStatusService                   │  │
│  │    (VpnService)           │   │    (Service)                              │  │
│  │                           │   │                                           │  │
│  │  implements:              │   │  implements:                              │  │
│  │  · StateListener          │   │  · LogListener                            │  │
│  │  · ByteCountListener      │   │  · ByteCountListener                      │  │
│  │  · IOpenVPNServiceInternal│   │  · StateListener                          │  │
│  │                           │   │  · ProfileNotifyListener                  │  │
│  │  AIDL Stub:               │   │                                           │  │
│  │  ┌──────────────────────┐ │   │  AIDL Stub:                               │  │
│  │  │IOpenVPNService       │ │   │  ┌─────────────────────────────────────┐  │  │
│  │  │Internal.Stub (mBinder│ │   │  │IServiceStatus.Stub (mBinder)        │  │  │
│  │  │  protect()           │ │   │  │  registerStatusCallback() → Pipe   │  │  │
│  │  │  stopVPN()           │ │   │  │  unregisterStatusCallback()        │  │  │
│  │  │  userPause()         │ │   │  │  getLastConnectedVPN()             │  │  │
│  │  │  challengeResponse() │ │   │  │  setCachedPassword()               │  │  │
│  │  │  addAllowedExtApp()  │ │   │  │  getTrafficHistory()               │  │  │
│  │  │  isAllowedExtApp()   │ │   │  │  notifyProfileVersionChanged()     │  │  │
│  │  └──────────────────────┘ │   │  └─────────────────────────────────────┘  │  │
│  └──────────┬────────────────┘   │                                           │  │
│             │                    │  Callbacks:                                │  │
│             │                    │  RemoteCallbackList<IStatusCallbacks>     │  │
│             │                    │  · newLogItem()                            │  │
│             │                    │  · updateStateString()                     │  │
│             │                    │  · updateByteCount()                       │  │
│             │                    │  · connectedVPN()                          │  │
│             │                    │  · notifyProfileVersionChanged()           │  │
│             │                    └──────────────────────┬────────────────────┘  │
│             │                                           │                     │
│             │         ┌─────────────────────────────────┘                     │
│             │         │  VpnStatus (观察者通知)                                 │
│             │         │                                                       │
│  ┌──────────▼─────────▼──────────────────┐                                     │
│  │    ExternalOpenVPNService              │                                     │
│  │    (Service)                           │                                     │
│  │                                        │                                     │
│  │  implements: StateListener             │                                     │
│  │                                        │                                     │
│  │  binds → OpenVPNService                │                                     │
│  │  (IOpenVPNServiceInternal)             │                                     │
│  │                                        │                                     │
│  │  AIDL Stub:                            │                                     │
│  │  ┌──────────────────────────────────┐  │                                     │
│  │  │IOpenVPNAPIService.Stub (mBinder) │  │                                     │
│  │  │  getProfiles()                   │  │                                     │
│  │  │  startProfile()                  │  │                                     │
│  │  │  startVPN()                      │  │                                     │
│  │  │  addNewVPNProfile()              │  │                                     │
│  │  │  removeProfile()                 │  │                                     │
│  │  │  disconnect() → mService.stopVPN │  │                                     │
│  │  │  pause() → mService.userPause    │  │                                     │
│  │  │  resume() → mService.userPause   │  │                                     │
│  │  │  protectSocket() → mService.protect│ │                                    │
│  │  │  prepare() / prepareVPNService() │  │                                     │
│  │  │  registerStatusCallback()        │  │                                     │
│  │  │  set/getDefaultProfile()         │  │                                     │
│  │  └──────────────────────────────────┘  │                                     │
│  │                                        │                                     │
│  │  Callbacks:                            │                                     │
│  │  RemoteCallbackList<IOpenVPNStatusCallback>                                  │
│  │  · newStatus(uuid, state, message, level)                                    │
│  └────────────────────────────────────────┘                                     │
│                                                                                 │
│  ┌──────────────────────────┐   ┌──────────────────────────────────────────┐   │
│  │    keepVPNAlive           │   │    VpnStatus (全局单例)                   │   │
│  │    (JobService)           │   │                                           │   │
│  │    implements:            │   │    Listeners:                             │   │
│  │    · StateListener        │   │    · OpenVPNService (State, ByteCount)    │   │
│  │                           │   │    · OpenVPNStatusService (State,Log,Byte)│   │
│  │    checks: isVPNActive()  │   │    · ExternalOpenVPNService (State)       │   │
│  │    restarts if inactive   │   │    · keepVPNAlive (State)                │   │
│  └──────────────────────────┘   └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
           │                    │                           │
           │ IOpenVPNService    │ IServiceStatus            │ IOpenVPNAPIService
           │ Internal           │ + IStatusCallbacks        │ + IOpenVPNStatusCallback
           │                    │                           │
           ▼                    ▼                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              UI 进程 (默认)                                     │
│                                                                                 │
│  ┌────────────────────────┐   ┌────────────────────────────────────────────┐   │
│  │  DisconnectVPN         │   │  StatusListener                             │   │
│  │  (Activity)            │   │  (初始化于 ICSOpenVPNApplication)           │   │
│  │                        │   │                                             │   │
│  │  binds → OpenVPNService│   │  binds → OpenVPNStatusService              │   │
│  │  (IOpenVPNService      │   │  (IServiceStatus)                          │   │
│  │   Internal)            │   │                                             │   │
│  │  → stopVPN()           │   │  注册 IStatusCallbacks:                    │   │
│  │  → userPause()         │   │  · newLogItem → VpnStatus.newLogItem()     │   │
│  └────────────────────────┘   │  · updateStateString → VpnStatus.update... │   │
│                                │  · updateByteCount → VpnStatus.update...   │   │
│  ┌────────────────────────┐   │  · connectedVPN → VpnStatus.setConnected...│   │
│  │  RemoteAction          │   │  · notifyProfileVersionChanged → PM.not...│   │
│  │  (Activity)            │   │                                             │   │
│  │  binds → OpenVPNService│   │  + 读取历史日志管道                         │   │
│  │  (IOpenVPNService      │   └─────────────────────────────────────────────┘   │
│  │   Internal)            │                                                     │
│  │  → stopVPN()           │   ┌─────────────────────────────────────────────┐   │
│  │  → userPause()         │   │  UI 组件                                      │   │
│  └────────────────────────┘   │  · MainActivity                              │   │
│                                │  · LogFragment ← VpnStatus (LogListener)     │   │
│  ┌────────────────────────┐   │  · GraphFragment ← VpnStatus (ByteCount)     │   │
│  │  OpenVPNTileService    │   │  · VPNPreferences                            │   │
│  │  (Quick Settings Tile) │   └─────────────────────────────────────────────┘   │
│  │  binds → OpenVPNService│                                                     │
│  │  (IOpenVPNService      │                                                     │
│  │   Internal)            │   ┌─────────────────────────────────────────────┐   │
│  │  → stopVPN()           │   │  第三方 App                                   │   │
│  └────────────────────────┘   │  (如 remoteExample)                           │   │
│                                │                                               │   │
│                                │  binds → ExternalOpenVPNService              │   │
│                                │  (IOpenVPNAPIService)                        │   │
│                                │                                               │   │
│                                │  注册 IOpenVPNStatusCallback:                │   │
│                                │  · newStatus(uuid, state, message, level)    │   │
│                                └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 完整调用流程

### 7.1 用户发起 VPN 连接

```
用户点击"连接"
    │
    ▼
LaunchVPN Activity
    ├── 解析 Intent 中的 profile UUID
    ├── 调用 VpnService.prepare() 请求 VPN 权限
    │   └── 如果未授权 → 系统弹出 VPN 权限对话框
    │       └── onActivityResult → 继续启动
    ├── 检查是否需要用户输入密码 (needUserPWInput)
    │   └── 如果需要 → 弹出密码对话框
    │       └── 用户输入后 → 通过 IServiceStatus.setCachedPassword() 缓存密码
    │           └── (跨进程：UI → OpenVPNStatusService → PasswordCache)
    └── VPNLaunchHelper.startOpenVpn(profile, context, reason, replace)
        ├── 构建 Intent: action=START_SERVICE, extra=profileUUID, profileVersion
        └── context.startForegroundService(intent)
            │
            ▼
        OpenVPNService.onStartCommand(intent)
```

### 7.2 OpenVPNService 启动与配置

```
OpenVPNService.onStartCommand(intent)
    ├── VpnStatus.addStateListener(this)     // 注册状态监听
    ├── VpnStatus.addByteCountListener(this) // 注册流量监听
    ├── 处理 PAUSE_VPN / RESUME_VPN action
    ├── showNotification() // 显示前台通知
    └── mCommandHandler.post(() -> startOpenVPN(intent, startId))
        │
        ▼
    startOpenVPN(intent, startId)
        ├── fetchVPNProfile(intent)
        │   ├── 从 Intent 获取 profileUUID 和 profileVersion
        │   ├── ProfileManager.get(context, uuid, version, timeout)
        │   │   └── 等待配置版本同步（跨进程可能延迟）
        │   └── 如果 intent 为 null → 使用上次连接的配置或 always-on 配置
        ├── checkVPNPermission(profile)
        │   └── VpnService.prepare() == null → 已授权
        │       └── 否则 → 显示权限请求通知
        ├── ProfileManager.setConnectedVpnProfile(this, vp)
        ├── VpnStatus.setConnectedVPNProfile(vp.getUUIDString())
        │   └── 通知所有 StateListener（包括 OpenVPNStatusService → UI 进程）
        ├── keepVPNAlive.scheduleKeepVPNAliveJobService(this, vp)
        ├── VPNLaunchHelper.buildOpenvpnArgv(this)
        │   └── 返回 ["libovpnexec.so路径", "--config", "stdin"]
        ├── stopOldOpenVPNProcess() // 中断旧的管理线程和进程线程
        │
        ├── [OpenVPN 2.x 路径]
        │   ├── new OpenVpnManagementThread(profile, this)
        │   ├── openManagementInterface() → 创建 Unix Domain Socket
        │   └── new Thread(managementThread).start()
        │
        ├── [OpenVPN 3.x 路径]
        │   └── new OpenVPNThreadv3(profile, this)
        │
        ├── profile.writeConfigFileOutput() → 写入配置到进程 stdin
        ├── registerDeviceStateReceiver() // 注册网络/屏幕状态监听
        └── VpnStatus.updateStateString("CONNECTING", ...)
```

### 7.3 原生 OpenVPN 进程启动与管理接口交互

```
OpenVPN 进程启动（由 OpenVPNThread 或 OpenVPNThreadv3 管理）
    │
    ├── 原生进程连接到 mgmtsocket
    │
    ▼
OpenVpnManagementThread.run()
    ├── mServerSocket.accept() // 等待 OpenVPN 连接
    ├── 发送初始命令:
    │   ├── "version 3\n"         // 管理接口版本
    │   ├── "hold release\n"      // 释放 hold 状态
    │   ├── "bytecount 2\n"       // 启用流量统计
    │   └── "state on\n"          // 启用状态通知
    │
    └── 循环读取并处理命令:
        │
        ├── >PASSWORD:Need='Auth' → 从 PasswordCache 获取用户名/密码
        │   └── managmentCommand("password Auth <user>\npassword Auth <pass>")
        │
        ├── >NEED-OK:OPENTUN → 调用 OpenVPNService.openTun()
        │   └── 创建 TUN 设备，通过 ancillary data 发送 FD
        │
        ├── >NEED-OK:DNSSERVER → tunConfig.mDnslist.add(dns)
        │
        ├── >NEED-OK:ROUTE → tunConfig.mRoutes.addIP(route, true)
        │
        ├── >NEED-OK:IFCONFIG → tunConfig.setLocalIP()/setMtu()
        │
        ├── >STATE:timestamp,state,info,... → VpnStatus.updateStateString()
        │   └── 触发观察者通知链:
        │       ├── OpenVPNService.updateState() → 更新通知栏
        │       ├── OpenVPNStatusService.updateState() → 通过 AIDL 转发到 UI
        │       └── ExternalOpenVPNService.updateState() → 通过 AIDL 转发到外部 App
        │
        ├── >BYTECOUNT:in,out → VpnStatus.updateByteCount(in, out)
        │   └── 触发观察者通知链（同上）
        │
        ├── >LOG:timestamp,level,message → VpnStatus.logMessageOpenVPN()
        │   └── 触发 LogListener 通知链
        │
        ├── >PROXY: → ProxyDetection 检测代理或使用配置中的代理
        │
        ├── >PK_SIGN: → ExtAuthHelper 调用外部证书 App 签名
        │
        ├── >HOLD: → 等待后自动 releaseHold (reconnect 场景)
        │
        ├── >INFOMSG:OPEN_URL: → OpenVPNService.trigger_sso() (浏览器 SSO)
        │   >INFOMSG:CR_TEXT: → trigger_sso() (挑战-响应)
        │   >INFOMSG:WEB_AUTH: → trigger_sso() (网页认证)
        │
        └── PROTECTFD: → mOpenVPNService.protect(fd) (VpnService.protect)
```

### 7.4 TUN 设备建立

当 OpenVPN 进程需要 TUN 设备时，发送 `>NEED-OK:OPENTUN` 命令：

```
>NEED-OK:OPENTUN
    │
    ▼
OpenVpnManagementThread.processNeedCommand("OPENTUN,...")
    ├── OpenVPNService.openTun()
    │   ├── VpnService.Builder builder = new VpnService.Builder()
    │   ├── builder.addAddress(localIP, netmask)        // 从 IFCONFIG 获取
    │   ├── builder.addDnsServer(dns)                    // 从 DNSSERVER 获取
    │   ├── builder.addRoute(route)                      // 从 ROUTE 获取
    │   ├── builder.setMtu(mtu)                          // 从 IFCONFIG 获取
    │   ├── builder.addDisallowedApplication()/addAllowedApplication()  // 用户配置
    │   ├── builder.setSession(profileName)
    │   ├── builder.establish() → ParcelFileDescriptor (TUN FD)
    │   └── 返回 TUN FD
    │
    └── 通过 Unix Domain Socket ancillary data 将 FD 传给 OpenVPN 进程
        └── managmentCommand("need-ok OPENTUN TUN_MTU_SIZE/65535 ...")
```

### 7.5 状态传播机制

状态传播是项目的核心设计，涉及两个进程和多个 AIDL 接口：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    :openvpn 进程                                        │
│                                                                         │
│  OpenVPN 进程                                                           │
│    │ >STATE:unix_timestamp,CONNECTED,...                                │
│    │ >BYTECOUNT:12345,67890                                             │
│    │ >LOG:timestamp,N,Connection established                           │
│    ▼                                                                    │
│  OpenVpnManagementThread                                                │
│    │ processState() → VpnStatus.updateStateString("CONNECTED", ...)    │
│    │ processByteCount() → VpnStatus.updateByteCount(12345, 67890)      │
│    │ processLogMessage() → VpnStatus.logMessageOpenVPN(...)            │
│    ▼                                                                    │
│  VpnStatus (单例，观察者分发)                                             │
│    ├── updateStateString() → 通知所有 StateListener:                     │
│    │   ├── OpenVPNService.updateState()                                 │
│    │   │   └── 更新通知栏、发送 VPN_STATUS 广播                          │
│    │   │                                                                │
│    │   ├── OpenVPNStatusService.updateState()                           │
│    │   │   └── mHandler.obtainMessage(SEND_NEW_STATE)                   │
│    │   │       → OpenVPNStatusHandler.handleMessage()                    │
│    │   │           → 遍历 mCallbacks (RemoteCallbackList)                │
│    │   │               → IStatusCallbacks.updateStateString()  ──────────┼──►
│    │   │                                                                │
│    │   └── ExternalOpenVPNService.updateState()                         │
│    │       └── mHandler.obtainMessage(SEND_TOALL)                       │
│    │           → 遍历 mCallbacks (RemoteCallbackList)                    │
│    │               → IOpenVPNStatusCallback.newStatus()  ───────────────────►
│    │                                                                    │
│    ├── updateByteCount() → 通知所有 ByteCountListener:                   │
│    │   ├── OpenVPNService.updateByteCount() → 通知栏流量信息             │
│    │   └── OpenVPNStatusService.updateByteCount()                       │
│    │       └── mHandler → IStatusCallbacks.updateByteCount()  ──────────┼──►
│    │                                                                    │
│    └── newLogItem() → 通知所有 LogListener:                              │
│        ├── OpenVPNStatusService.newLog()                                │
│        │   └── mHandler → IStatusCallbacks.newLogItem()  ───────────────┼──►
│        └── LogFileHandler → 写入日志文件                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                                    │
                ┌───────────────────────────────────┘
                │
                ▼ IStatusCallbacks (AIDL, oneway)
┌─────────────────────────────────────────────────────────────────────────┐
│                    UI 进程                                              │
│                                                                         │
│  StatusListener.mCallback                                               │
│    ├── newLogItem(item) → VpnStatus.newLogItem(item)                   │
│    ├── updateStateString(...) → VpnStatus.updateStateString(...)       │
│    ├── updateByteCount(in, out) → VpnStatus.updateByteCount(in, out)   │
│    ├── connectedVPN(uuid) → VpnStatus.setConnectedVPNProfile(uuid)     │
│    └── notifyProfileVersionChanged(uuid, ver) → PM.notifyProfileVersion│
│                                                                         │
│  VpnStatus (UI 进程实例)                                                 │
│    └── 通知 UI 组件 (LogFragment, GraphFragment, TileService 等)        │
│                                                                         │
│                ┌───────────────────────────────────────┐                │
│                │ IOpenVPNStatusCallback (AIDL, oneway) │                │
│                └───────────────────────────────────────┘                │
│                    │                                                     │
│                    ▼                                                     │
│  第三方 App 的回调实现                                                    │
│    └── newStatus(uuid, state, message, level) → 更新 App UI             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.6 VPN 断开流程

```
用户点击"断开" / 通知栏断开按钮 / 外部 App 调用 disconnect()
    │
    ▼
DisconnectVPN Activity / IOpenVPNServiceInternal.stopVPN() / IOpenVPNAPIService.disconnect()
    │
    ▼
OpenVPNService.stopVPN(replaceConnection)
    │
    ▼
OpenVPNManagement.stopVPN()
    │ managmentCommand("signal SIGINT\n")
    │
    ▼
OpenVPN 进程接收 SIGINT
    │
    ▼
OpenVPN 进程退出，发送 >STATE:...EXITING
    │
    ▼
OpenVpnManagementThread 检测到 socket 关闭或进程结束
    │
    ▼
VpnStatus.updateStateString("NOPROCESS", LEVEL_NOTCONNECTED)
    │
    ▼
OpenVPNService.openvpnStopped() → endVpnService()
    ├── VpnStatus.removeByteCountListener(this)
    ├── unregisterDeviceStateReceiver()
    ├── ProfileManager.setConntectedVpnProfileDisconnected()
    ├── keepVPNAlive.unscheduleKeepVPNAliveJobService()（非 always-on 时）
    ├── stopForeground()
    └── stopSelf()
```

### 7.7 外部 App 控制 VPN 流程

```
第三方 App 启动 VPN 连接:
    │
    ├── bindService → ExternalOpenVPNService
    │
    ├── prepare(packageName)
    │   ├── 检查 ExternalAppDatabase 白名单
    │   │   └── 已授权 → 返回 null
    │   └── 未授权 → 返回 ConfirmDialog Intent
    │       └── 用户确认 → 添加到白名单
    │
    ├── prepareVPNService()
    │   └── VpnService.prepare() → 返回权限 Intent 或 null
    │
    ├── startProfile(uuid) / startVPN(inlineConfig)
    │   ├── checkOpenVPNPermission()
    │   ├── 解析/获取 VpnProfile
    │   ├── 检查 checkProfile()
    │   ├── 如果需要密码 → NotifyHelper.showLaunchNotify() 弹通知
    │   └── 否则 → VPNLaunchHelper.startOpenVpn()
    │
    └── registerStatusCallback(callback)
        └── ExternalOpenVPNService 通过 VpnStatus.StateListener 广播状态

第三方 App 断开 VPN:
    │
    ├── disconnect()
    │   └── mService.stopVPN(false)  // IOpenVPNServiceInternal.stopVPN()
    │
    └── unregisterStatusCallback(callback)
```

### 7.8 开机自启动流程

```
系统启动完成 (BOOT_COMPLETED) 或 App 更新 (MY_PACKAGE_REPLACED)
    │
    ▼
OnBootReceiver.onReceive()
    │
    ├── 检查 "restartvpnonboot" 偏好设置
    │
    └── 如果启用 → VPNLaunchHelper.startOpenVpn(profile, context, "on boot", false)
        │
        ▼
    OpenVPNService.onStartCommand()
        └── 正常 VPN 启动流程
```

---

## 8. 进程间通信架构图

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                        │
│                                   :openvpn 进程                                        │
│                                                                                        │
│  ┌─────────────────┐    ┌──────────────────────┐    ┌──────────────────────────────┐   │
│  │  OpenVPNService  │    │ OpenVPNStatusService  │    │ ExternalOpenVPNService       │   │
│  │                  │    │                       │    │                              │   │
│  │ IOpenVPNService  │    │ IServiceStatus        │    │ IOpenVPNAPIService           │   │
│  │ Internal.Stub    │    │ .Stub                 │    │ .Stub                        │   │
│  │                  │    │                       │    │                              │   │
│  │ protect() ───────────► VpnService.protect() │    │ getProfiles() ──────────────►│   │
│  │ stopVPN() ──────────► management.stopVPN()  │    │ startProfile() ─────────────►│   │
│  │ userPause() ───────► DeviceStateReceiver    │    │ disconnect() ───► mService   │   │
│  │ challengeResponse()► management.sendCRResp  │    │ pause() ────────► mService   │   │
│  │                  │    │                       │    │ resume() ───────► mService   │   │
│  │                  │    │ registerCallback()    │    │ protectSocket() ► mService   │   │
│  │                  │    │  → 返回日志 Pipe      │    │                              │   │
│  │                  │    │                       │    │ IOpenVPNStatusCallback       │   │
│  │                  │    │ IStatusCallbacks      │    │ RemoteCallbackList           │   │
│  │                  │    │ RemoteCallbackList    │    │                              │   │
│  └──────┬───────────┘    └──────────┬───────────┘    │           │                  │   │
│         │                           │                └───────────┼──────────────────┘   │
│         │                           │                            │                      │
│         │                           │  VpnStatus (观察者模式)      │                      │
│         │                           │                            │                      │
│         │  ┌────────────────────────▼────────────────────────────▼──────────────────┐  │
│         │  │                        VpnStatus                                       │  │
│         │  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  ┌─────────────┐ │  │
│         │  │  │ LogListeners │  │StateListeners│  │ByteCountLstn│  │ProfileLstnrs│ │  │
│         │  │  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘  └──────┬──────┘ │  │
│         │  └─────────┼─────────────────┼─────────────────┼────────────────┼────────┘  │
│         │            │                 │                 │                │            │
│         │            ▼                 ▼                 ▼                ▼            │
│         │   OpenVPNStatusService   OpenVPNService   OpenVPNStatus    OpenVPNStatus    │
│         │   .newLog()             .updateState()   Service          Service           │
│         │                                              .updateByte()   .notifyProfile()  │
│         │   ExternalOpenVPN                                      ExternalOpenVPN     │
│         │   Service.updateState()                               Service (state)     │
│         │                                                                            │
│  ┌──────┴──────────────────────────────────────────────────────────────────────────┐  │
│  │  OpenVpnManagementThread                                                        │  │
│  │  ┌─────────────┐     Unix Domain Socket     ┌──────────────┐                    │  │
│  │  │  Java 层     │◄──────────────────────────►│ OpenVPN 进程  │                    │  │
│  │  │  (管理接口)  │     cacheDir/mgmtsocket    │  (C/C++)     │                    │  │
│  │  └─────────────┘                              └──────────────┘                    │  │
│  │  ↑ processCommand()                                                              │  │
│  │  │ >STATE → VpnStatus.updateStateString()                                        │  │
│  │  │ >LOG → VpnStatus.logMessageOpenVPN()                                          │  │
│  │  │ >BYTECOUNT → VpnStatus.updateByteCount()                                      │  │
│  │  │ >NEED-OK:OPENTUN → OpenVPNService.openTun()                                   │  │
│  │  │ >PASSWORD → PasswordCache                                                     │  │
│  │  │ PROTECTFD → VpnService.protect()                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
         │                          │                              │
         │ IOpenVPNService          │ IServiceStatus               │ IOpenVPNAPIService
         │ Internal (AIDL)          │ + IStatusCallbacks (AIDL)    │ + IOpenVPNStatusCallback
         │                          │                              │
         ▼                          ▼                              ▼
┌──────────────────┐   ┌──────────────────────────┐   ┌──────────────────────────┐
│  DisconnectVPN   │   │  StatusListener           │   │  第三方 App               │
│  RemoteAction    │   │  (UI 进程)                │   │  (如 remoteExample)       │
│  OpenVPNTile     │   │                            │   │                          │
│  Service         │   │  IStatusCallbacks.Stub     │   │  IOpenVPNStatusCallback  │
│                  │   │  ┌──────────────────────┐  │   │  ┌────────────────────┐  │
│  stopVPN()       │   │  │ newLogItem →         │  │   │  │ newStatus →        │  │
│  userPause()     │   │  │   VpnStatus.newLog   │  │   │  │   更新 App UI      │  │
│  protect()       │   │  │ updateStateString →  │  │   │  └────────────────────┘  │
│                  │   │  │   VpnStatus.update   │  │   │                          │
│                  │   │  │ updateByteCount →    │  │   │  + 读取历史日志管道       │
│                  │   │  │   VpnStatus.update   │  │   │                          │
│                  │   │  │ connectedVPN →       │  │   └──────────────────────────┘
│                  │   │  │   VpnStatus.set      │  │
│                  │   │  └──────────────────────┘  │
│                  │   │                            │
│                  │   │  + 读取历史日志管道         │
│                  │   └──────────────────────────┘
└──────────────────┘
```

---

## 9. 状态机与连接状态

### OpenVPN 原生状态 → ConnectionStatus 映射

```
OpenVPN 状态字符串           ConnectionStatus              通知栏图标
─────────────────────────────────────────────────────────────────────────
NOPROCESS                  → LEVEL_NOTCONNECTED           → ic_stat_vpn_offline
DISCONNECTED               → LEVEL_NOTCONNECTED           → ic_stat_vpn_offline
EXITING                    → LEVEL_NOTCONNECTED           → ic_stat_vpn_offline

CONNECTING                 → LEVEL_CONNECTING_NO_SERVER   → ic_stat_vpn_outline
WAIT                       → LEVEL_CONNECTING_NO_SERVER   → ic_stat_vpn_outline
RECONNECTING               → LEVEL_CONNECTING_NO_SERVER   → ic_stat_vpn_outline
RESOLVE                    → LEVEL_CONNECTING_NO_SERVER   → ic_stat_vpn_outline
TCP_CONNECT                → LEVEL_CONNECTING_NO_SERVER   → ic_stat_vpn_outline

AUTH                       → LEVEL_CONNECTING_SERVER_REPLIED → ic_stat_vpn_empty_halo
GET_CONFIG                 → LEVEL_CONNECTING_SERVER_REPLIED → ic_stat_vpn_empty_halo
ASSIGN_IP                  → LEVEL_CONNECTING_SERVER_REPLIED → ic_stat_vpn_empty_halo
ADD_ROUTES                 → LEVEL_CONNECTING_SERVER_REPLIED → ic_stat_vpn_empty_halo
AUTH_PENDING               → LEVEL_CONNECTING_SERVER_REPLIED → ic_stat_vpn_empty_halo

CONNECTED                  → LEVEL_CONNECTED              → ic_stat_vpn

(认证失败)                  → LEVEL_AUTH_FAILED            → ic_stat_vpn_offline
(用户暂停/屏幕关闭)         → LEVEL_VPNPAUSED             → ic_media_pause
(无网络)                    → LEVEL_NONETWORK             → ic_stat_vpn_offline
(等待用户输入)              → LEVEL_WAITING_FOR_USER_INPUT → ic_stat_vpn_outline
```

### 连接状态转换图

```
                              ┌──────────────────────┐
                              │     NOPROCESS        │
                              │  (LEVEL_NOTCONNECTED)│
                              └──────────┬───────────┘
                                         │ startOpenVPN()
                                         ▼
                              ┌──────────────────────┐
                              │     CONNECTING       │
                              │ (LEVEL_CONNECTING    │
                              │  _NO_SERVER_REPLY)   │
                              └──────────┬───────────┘
                                         │
                         ┌───────────────┼───────────────┐
                         ▼               ▼               ▼
                ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                │    WAIT      │ │   RESOLVE    │ │ TCP_CONNECT  │
                │(NO_SERVER    │ │(NO_SERVER    │ │(NO_SERVER    │
                │ _REPLY)      │ │ _REPLY)      │ │ _REPLY)      │
                └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
                       │                │                │
                       └────────────────┼────────────────┘
                                        ▼
                              ┌──────────────────────┐
                              │       AUTH           │
                              │ (LEVEL_CONNECTING    │
                              │  _SERVER_REPLIED)    │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │    GET_CONFIG        │
                              │ (LEVEL_CONNECTING    │
                              │  _SERVER_REPLIED)    │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │    ASSIGN_IP         │
                              │ (LEVEL_CONNECTING    │
                              │  _SERVER_REPLIED)    │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │    ADD_ROUTES        │
                              │ (LEVEL_CONNECTING    │
                              │  _SERVER_REPLIED)    │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │     CONNECTED        │◄──────────────────┐
                              │  (LEVEL_CONNECTED)   │                   │
                              └──────────┬───────────┘                   │
                                         │                               │
                         ┌───────────────┼───────────────┐               │
                         ▼               ▼               ▼               │
                ┌──────────────┐ ┌──────────────┐ ┌──────────────┐      │
                │  DISCONNECTED│ │   EXITING    │ │ RECONNECTING │──────┘
                │(NOTCONNECTED)│ │(NOTCONNECTED)│ │(NO_SERVER    │
                └──────────────┘ └──────────────┘ │ _REPLY)      │
                                    ▲              └──────────────┘
                                    │
                              ┌─────┴──────┐
                              │ AUTH_FAILED│
                              │(AUTH_FAILED│
                              └────────────┘

         ┌──────────────────────────────────────────────┐
         │          暂停状态 (用户暂停/屏幕关闭/无网络)    │
         │                                              │
         │  USERPAUSE  → LEVEL_VPNPAUSED               │
         │  SCREENOFF  → LEVEL_VPNPAUSED               │
         │  NONETWORK  → LEVEL_NONETWORK               │
         │                                              │
         │  恢复条件: 用户恢复/屏幕打开/网络恢复           │
         └──────────────────────────────────────────────┘
```

---

## 10. 关键设计决策与权衡

### 10.1 为什么用两个 Service（OpenVPNService + OpenVPNStatusService）而不是一个？

- **OpenVPNService** 继承 `VpnService`，其 `onBind()` 有两种行为：action 为 `START_SERVICE` 时返回自定义 binder，否则返回 VpnService 默认 binder
- **OpenVPNStatusService** 是纯状态桥接服务，职责单一，生命周期独立
- 分离关注点：VPN 引擎控制与状态广播解耦，避免 VpnService 的 binder 冲突

### 10.2 为什么 StatusListener 判断 `queryLocalInterface`？

```java
if (service.queryLocalInterface("de.blinkt.openvpn.core.IServiceStatus") == null) {
    // 跨进程：注册回调 + 读取历史日志管道
} else {
    // 同进程：直接初始化日志缓存
}
```

当 `StatusListener` 和 `OpenVPNStatusService` 在同一进程时（如 skeleton 变体），不需要跨进程通信，可以直接操作 VpnStatus 的日志缓存。

### 10.3 为什么 OpenVPNStatusHandler 是 static 的？

```java
private static final OpenVPNStatusHandler mHandler = new OpenVPNStatusHandler();
```

- Handler 是 static 的，使用 `WeakReference` 持有 Service
- 防止 Service 重建时丢失消息队列中的待处理事件
- 避免内存泄漏

### 10.4 为什么 IStatusCallbacks 的方法都是 oneway？

所有回调方法标记为 `oneway`，确保：
- 不会阻塞 `:openvpn` 进程的 VpnStatus 事件分发
- 即使 UI 进程处理慢，也不会影响 VPN 引擎的实时状态更新
- 事件可能丢失（oneway 不保证送达），但对状态更新场景可接受

### 10.5 历史日志为什么用 Pipe 而不是 AIDL 直接传递？

`registerStatusCallback()` 返回 `ParcelFileDescriptor` 管道而非直接在 AIDL 方法中返回日志列表：
- 日志缓冲区可能很大（最多 1000 条），AIDL 事务有 1MB 限制
- 管道传输是流式的，不受 AIDL 事务大小限制
- 读取在独立线程中进行，不阻塞主线程

### 10.6 为什么 ExternalOpenVPNService 要绑定 OpenVPNService？

`ExternalOpenVPNService` 需要调用 `OpenVPNService` 的 `protect()`, `stopVPN()`, `userPause()` 等方法，这些方法只能通过 `IOpenVPNServiceInternal` AIDL 接口访问。直接绑定避免了通过 `startService` +广播的复杂通信方式。

### 10.7 密码缓存跨进程传递

用户在 UI 进程输入密码后，通过 `IServiceStatus.setCachedPassword(uuid, type, password)` 传递到 `:openvpn` 进程的 `PasswordCache`。当 OpenVPN 通过管理接口请求密码时（`>PASSWORD:Need='Auth'`），`OpenVpnManagementThread` 从 `PasswordCache` 获取密码并发送给 OpenVPN。

### 10.8 配置版本同步

UI 进程修改配置后，`ProfileManager` 保存配置并递增版本号。通过 `VpnStatus.notifyProfileVersionChanged()` → `StatusListener` → `IServiceStatus.notifyProfileVersionChanged()` → `OpenVPNStatusService` → `ProfileManager.notifyProfileVersionChanged()` 在 `:openvpn` 进程中通知更新。`OpenVPNService.fetchVPNProfile()` 会等待配置版本同步（最多 10 秒），确保使用最新的配置启动 VPN。

---

## 11. 代码解析

### 11.1 SignaturePadding 枚举 — 外部 PKI 签名填充模式

**定义位置**: `core/OpenVPNManagement.java:19-23`

```java
enum SignaturePadding {
    RSA_PKCS1_PSS_PADDING,  // RSA-PSS 概率签名方案填充
    RSA_PKCS1_PADDING,      // RSA PKCS#1 v1.5 填充
    NO_PADDING              // 无填充（原始签名）
}
```

这个枚举定义了 OpenVPN 外部 PKI（Public Key Infrastructure）签名时使用的 **RSA 填充模式**。当 VPN 连接使用外部证书（Android KeyChain 或外部 App 提供的私钥）进行 TLS 认证时，OpenVPN 原生进程会请求 Java 层对数据进行签名，Java 层需要知道使用哪种填充方式。

#### 三种填充模式详解

| 枚举值 | 含义 | 适用场景 | 密码学说明 |
|--------|------|----------|------------|
| `RSA_PKCS1_PSS_PADDING` | RSASSA-PSS（概率签名方案） | 现代安全要求，OpenVPN 2.6+ 默认 | 将消息摘要加盐后填充，每次签名结果不同，具有可证明安全性证明。需配合 `saltlen` 和 `hashalg` 参数使用 |
| `RSA_PKCS1_PADDING` | RSASSA-PKCS1-v1_5（确定性签名） | 传统兼容模式，旧版 OpenVPN/服务器 | 将 DigestInfo 结构体按 PKCS#1 v1.5 格式填充，确定性签名（相同输入总是产生相同签名），存在潜在安全风险但仍广泛兼容 |
| `NO_PADDING` | 无填充 | ECDSA 椭圆曲线签名、原始 RSA 操作 | 直接对数据进行签名/加密，不做任何填充。ECDSA 本身不需要填充；RSA 场景中用于已自行完成填充的数据 |

#### 签名请求的触发流程

```
OpenVPN 原生进程需要外部私钥签名
    │
    ├── [OpenVPN 2.x 路径]
    │   发送管理接口命令: >PK_SIGN:NC9t8IkYrj...,RSA_PKCS1_PSS_PADDING,hashalg=SHA256,saltlen=digest
    │   └── OpenVpnManagementThread.processSignCommand() 解析参数
    │
    └── [OpenVPN 3.x 路径]
        回调: ClientAPI_ExternalPKISignRequest.getAlgorithm() 返回 "RSA_PKCS1_PSS_PADDING"
        └── OpenVPNThreadv3.external_pki_sign_request() 映射算法字符串到枚举
```

**OpenVPN 2.x 管理接口路径**（`OpenVpnManagementThread.java:763-800`）：

```
>PK_SIGN:NC9t8IkYrjAQcCzc85zN0H5TvwfAUDwYkR4j2ga6fGw=,RSA_PKCS1_PSS_PADDING,hashalg=SHA256,saltlen=digest
         │                                   │                     │                │
         │ Base64 编码的待签名数据            │ 填充模式              │ 哈希算法        │ 盐长度
         └─ arguments[0]                     └─ 映射到枚举           └─ hashalg       └─ saltlen
```

解析逻辑：
```java
SignaturePadding padding = SignaturePadding.NO_PADDING;  // 默认无填充
String saltlen = "";
String hashalg = "";
boolean needsDigest = false;

for (int i = 1; i < arguments.length; i++) {
    String arg = arguments[i];
    if (arg.equals("RSA_PKCS1_PADDING"))
        padding = SignaturePadding.RSA_PKCS1_PADDING;
    else if (arg.equals("RSA_PKCS1_PSS_PADDING"))
        padding = SignaturePadding.RSA_PKCS1_PSS_PADDING;
    else if (arg.startsWith("saltlen="))
        saltlen = arg.substring(8);
    else if (arg.startsWith("hashalg="))
        hashalg = arg.substring(8);
    else if (arg.equals("data=message"))
        needsDigest = true;  // 需要先对原文做摘要再签名
}
```

**OpenVPN 3.x 回调路径**（`OpenVPNThreadv3.java:281-301`）：

```java
SignaturePadding padding;
switch (signreq.getAlgorithm()) {
    case "RSA_PKCS1_PSS_PADDING":  padding = SignaturePadding.RSA_PKCS1_PSS_PADDING; break;
    case "RSA_PKCS1_PADDING":      padding = SignaturePadding.RSA_PKCS1_PADDING;     break;
    case "RSA_NO_PADDING":         padding = SignaturePadding.NO_PADDING;            break;
    case "ECDSA":                  padding = SignaturePadding.NO_PADDING;            break;
    default: throw new IllegalArgumentException("Illegal padding in sign request");
}
```

注意：ECDSA 算法也映射到 `NO_PADDING`，因为椭圆曲线签名不需要 RSA 填充。

#### 签名执行路径

解析完成后，统一调用 `VpnProfile.getSignedData()`（`VpnProfile.java:1221-1234`），根据认证类型分两条路径：

**路径一：Android KeyChain 签名**（`VpnProfile.java:1270-1408`）

```
VpnProfile.getKeyChainSignedData(data, padding, saltlen, hashalg, needDigest)
    │
    ├── needDigest == true 或 密钥为 EC
    │   └── doDigestSign(privkey, data, padding, hashalg, saltlen)
    │       │
    │       ├── EC 密钥
    │       │   └── Signature.getInstance("SHA256withECDSA") 等标准 JCA 签名
    │       │
    │       ├── RSA_PKCS1_PSS_PADDING
    │       │   ├── API 23+ → Signature.getInstance("SHA256withRSA/PSS")
    │       │   │          + PSSParameterSpec(saltlen=digest)
    │       │   └── API < 23 → addPSSPadding() 手动添加 PSS 填充
    │       │                    → 递归调用 getKeyChainSignedData(NO_PADDING)
    │       │                    → Cipher.getInstance("RSA/ECB/NoPadding")
    │       │
    │       └── RSA_PKCS1_PADDING
    │           └── Signature.getInstance("SHA256withRSA") 等标准 JCA 签名
    │
    └── needDigest == false 且 密钥为 RSA
        │
        ├── RSA_PKCS1_PADDING → Cipher.getInstance("RSA/ECB/PKCS1PADDING")
        ├── NO_PADDING        → Cipher.getInstance("RSA/ECB/NoPadding")
        └── RSA_PKCS1_PSS_PADDING → 抛出异常
            （PSS 不能在没有摘要的情况下使用）
```

**路径二：外部 App 签名**（`VpnProfile.java:1236-1268`）

```
VpnProfile.getExtAppSignedData(data, padding, saltlen, hashalg, needDigest)
    │
    ├── 将 SignaturePadding 映射为 RsaPaddingType（AIDL Parcelable 传参）
    │   RSA_PKCS1_PSS_PADDING → RSAPSS_PADDING
    │   RSA_PKCS1_PADDING     → PKCS1_PADDING
    │   NO_PADDING            → NO_PADDING
    │
    ├── 打包 Bundle：paddingType, saltlen, hashalg, needsDigest
    │
    └── ExtAuthHelper.signData(context, authenticator, alias, data, extra)
        → 通过 AIDL 调用第三方 App 的 ExternalCertificateProvider
```

#### 三种填充模式的密码学对比

| 特性 | PKCS#1 v1.5 | PSS | No Padding |
|------|-------------|-----|------------|
| 安全性证明 | 无 | 有（可证明安全） | N/A（非完整签名方案） |
| 签名确定性 | 确定（相同输入→相同输出） | 概率（相同输入→不同输出） | 确定 |
| 盐值要求 | 无 | 需要 saltlen（通常 = digest 长度） | 无 |
| 兼容性 | 最广泛 | OpenVPN 2.6+ / TLS 1.3 倾向 | 仅特殊场景 |
| API 最低版本 | API 1 | API 23（`RSA/PSS`） | API 1 |
| API < 23 回退 | N/A | 手动 `addPSSPadding()` + `NoPadding` | N/A |

#### 关键源文件索引

| 文件 | 行号 | 内容 |
|------|------|------|
| `core/OpenVPNManagement.java` | 19-23 | `SignaturePadding` 枚举定义 |
| `core/OpenVpnManagementThread.java` | 763-800 | OpenVPN 2.x 签名命令解析（`processSignCommand`） |
| `ui/core/OpenVPNThreadv3.java` | 281-301 | OpenVPN 3.x 签名算法映射（`external_pki_sign_request`） |
| `VpnProfile.java` | 1221-1234 | `getSignedData()` — 统一签名入口，分发到 KeyChain 或外部 App |
| `VpnProfile.java` | 1236-1268 | `getExtAppSignedData()` — 外部 App 签名路径，映射到 `RsaPaddingType` |
| `VpnProfile.java` | 1270-1408 | `getKeyChainSignedData()` — KeyChain 签名路径，含 `doDigestSign` 和 Cipher 分支 |
| `VpnProfile.java` | 1310-1323 | `addPSSPadding()` — API < 23 的 PSS 手动填充回退 |
| `VpnProfile.java` | 1349-1391 | `doDigestSign()` — 带摘要的签名实现（ECDSA、PSS、PKCS1） |
| `VpnProfile.java` | 1406-1410 | `RsaPaddingType` — 外部 App AIDL 使用的填充类型枚举 |

### 11.2 OpenVPN 2.x 与 OpenVPN 3.x 双引擎架构

项目中 `useOpenVPN3` 变量控制着使用哪一代 OpenVPN 核心引擎。两代引擎在**进程模型、通信方式、TUN 建立、Socket 保护**等方面有根本性差异。

#### 11.2.1 架构选择机制

**决策入口**（`VpnProfile.java:227-233`）：

```java
public static boolean doUseOpenVPN3(Context c) {
    SharedPreferences prefs = Preferences.getDefaultSharedPreferences(c);
    boolean useOpenVPN3 = prefs.getBoolean("ovpn3", false);  // 用户偏好
    if (!BuildConfig.openvpn3)                                // 编译时开关
        useOpenVPN3 = false;
    return useOpenVPN3;
}
```

必须同时满足两个条件：
1. **`BuildConfig.openvpn3 == true`** — 编译时由 Gradle Product Flavor 决定
2. **用户偏好 `ovpn3 == true`** — 运行时由 SharedPreferences 控制

**Gradle Product Flavors**（`build.gradle.kts:115-136`）：

```
flavorDimensions = ["implementation", "ovpnimpl"]

productFlavors {
    ui       { dimension = "implementation" }      // 完整 UI
    skeleton { dimension = "implementation" }       // 精简 UI

    ovpn23 {                                        // OpenVPN 2+3 混合包
        dimension = "ovpnimpl"
        buildConfigField("boolean", "openvpn3", "true")
    }
    ovpn2 {                                         // 仅 OpenVPN 2.x 包
        dimension = "ovpnimpl"
        versionNameSuffix = "-o2"
        buildConfigField("boolean", "openvpn3", "false")
    }
}
```

| Flavor 组合 | 包含引擎 | `BuildConfig.openvpn3` | 用户可选 |
|-------------|----------|------------------------|----------|
| `uiOvpn23` | OpenVPN 2.x + 3.x | `true` | 是（SharedPreferences 切换） |
| `uiOvpn2` | 仅 OpenVPN 2.x | `false` | 否（强制 2.x） |
| `skeletonOvpn23` | OpenVPN 2.x + 3.x | `true` | 是 |
| `skeletonOvpn2` | 仅 OpenVPN 2.x | `false` | 否 |

**启动时的分支**（`OpenVPNService.java:701-740`）：

```java
boolean useOpenVPN3 = VpnProfile.doUseOpenVPN3(this);

if (!useOpenVPN3) {
    // OpenVPN 2.x: 创建 Unix Domain Socket 管理接口
    OpenVpnManagementThread ovpnManagementThread = new OpenVpnManagementThread(mProfile, this);
    if (ovpnManagementThread.openManagementInterface(this)) {
        Thread mSocketManagerThread = new Thread(ovpnManagementThread, "OpenVPNManagementThread");
        mSocketManagerThread.start();
        mManagement = ovpnManagementThread;
    }
}

Runnable processThread;
if (useOpenVPN3) {
    // OpenVPN 3.x: 通过反射实例化 OpenVPNThreadv3（仅在 ovpn23 flavor 中存在）
    OpenVPNManagement mOpenVPN3 = instantiateOpenVPN3Core();
    processThread = (Runnable) mOpenVPN3;
    mManagement = mOpenVPN3;
} else {
    // OpenVPN 2.x: 启动独立进程
    processThread = new OpenVPNThread(this, argv, nativeLibraryDirectory, tmpDir);
}

mProcessThread = new Thread(processThread, "OpenVPNProcessThread");
mProcessThread.start();

if (!useOpenVPN3) {
    // OpenVPN 2.x: 通过 stdin 写入配置文件
    mProfile.writeConfigFileOutput(this, ((OpenVPNThread) processThread).getOpenVPNStdin());
}
```

`instantiateOpenVPN3Core()` 使用反射加载（`OpenVPNService.java:786-795`）：

```java
private OpenVPNManagement instantiateOpenVPN3Core() {
    try {
        Class<?> cl = Class.forName("de.blinkt.openvpn.core.OpenVPNThreadv3");
        return (OpenVPNManagement) cl.getConstructor(OpenVPNService.class, VpnProfile.class)
                                     .newInstance(this, mProfile);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```

使用反射是因为 `OpenVPNThreadv3` 在 `ui` sourceSet 中，`ovpn2` flavor 编译时不存在该类，直接引用会导致编译错误。

#### 11.2.2 核心架构对比

| 维度 | OpenVPN 2.x | OpenVPN 3.x |
|------|-------------|-------------|
| **进程模型** | 独立子进程（`ProcessBuilder` 启动 `libovpnexec.so`） | 进程内线程（`Thread.run()` 中调用 JNI） |
| **原生库** | `libovpnexec.so`（OpenVPN 2.x C 代码） | `libovpn3.so`（OpenVPN 3.x C++ 核心 + SWIG 绑定） |
| **Java 入口** | `OpenVPNThread` implements `Runnable` | `OpenVPNThreadv3` extends `ClientAPI_OpenVPNClient` implements `Runnable`, `OpenVPNManagement` |
| **管理接口** | `OpenVpnManagementThread` — Unix Domain Socket 文本协议 | **无** — 通过 Java 虚拟方法回调直接交互 |
| **配置传递** | 写入子进程 stdin（`--config stdin`） | `ClientAPI_Config.setContent(configString)` |
| **TUN 建立** | `>NEED-OK:OPENTUN` → `openTun()` → FD 通过 `SCM_RIGHTS` 传回 | `tun_builder_establish()` 回调 → `openTun().detachFd()` 返回 int |
| **Socket 保护** | `PROTECTFD:` + `SCM_RIGHTS` 传 FD → `VpnService.protect()` | `socket_protect(socket, remote, ipv6)` 回调 → `VpnService.protect()` |
| **路由/DNS 配置** | `>NEED-OK:IFCONFIG/ROUTE/DNSSERVER` 文本命令 | `tun_builder_add_route()` / `tun_builder_set_dns_options()` 等回调 |
| **状态/日志** | `>STATE:` / `>LOG:` / `>BYTECOUNT:` 文本命令 | `event()` / `log()` 回调 + `pollStatus()` 定时轮询 |
| **外部 PKI 签名** | `>PK_SIGN:data,padding,hashalg,saltlen` 文本命令 | `external_pki_sign_request()` / `external_pki_cert_request()` 回调 |
| **密码/认证** | `>PASSWORD:Need 'Auth'` 文本命令 | `ClientAPI_ProvideCreds` 在 `setUserPW()` 中预设置 |
| **SSO/CR** | `>INFOMSG:OPEN_URL/CR_TEXT/WEB_AUTH` | `event("INFO", "OPEN_URL:...")` 事件 |
| **停止 VPN** | `signal SIGINT\n` 通过 UDS 发送 | `stop()` JNI 方法 |
| **重连** | `signal SIGUSR1\n` + `hold release\n` | `reconnect()` JNI 方法 |
| **网络切换** | `network-change\n` 通过 UDS 发送 | `reconnect(1)` JNI 方法 |
| **流量统计** | `>BYTECOUNT:in,out` 自动推送 | `pollStatus()` 每 2 秒调用 `transport_stats()` 主动轮询 |
| **日志解析** | 正则匹配 `LOG_PATTERN` 解析 flags/level | `log(ClientAPI_LogInfo)` 结构化回调 |

#### 11.2.3 OpenVPN 2.x 详细架构

**启动流程**：

```
OpenVPNService.startOpenVPN()
    │
    ├── VPNLaunchHelper.buildOpenvpnArgv()
    │   └── ["libovpnexec.so路径", "--config", "stdin"]
    │
    ├── OpenVpnManagementThread.openManagementInterface()
    │   └── 创建 Unix Domain Socket 服务端 (cacheDir/mgmtsocket)
    │       └── new LocalServerSocket → bind → listen → accept [阻塞等待]
    │
    ├── new OpenVPNThread(service, argv, nativeLibDir, tmpDir)
    │
    ├── new Thread(openVPNThread, "OpenVPNProcessThread").start()
    │   │
    │   └── OpenVPNThread.run()
    │       ├── ProcessBuilder(argv).start() → 启动独立子进程
    │       ├── 设置 LD_LIBRARY_PATH, TMPDIR 环境变量
    │       ├── 合并 stdout/stderr (redirectErrorStream=true)
    │       ├── 循环读取子进程 stdout → 正则解析日志
    │       │   └── VpnStatus.logMessageOpenVPN(logStatus, logLevel, msg)
    │       └── 进程退出 → VpnStatus.updateStateString("NOPROCESS")
    │
    ├── mProfile.writeConfigFileOutput(stdin)
    │   └── 将 OpenVPN 配置文本写入子进程 stdin
    │       包括: management <path>/mgmtsocket unix
    │             management-client
    │             management-query-passwords
    │             management-hold
    │
    └── [子进程连接 mgmtsocket]
        └── OpenVpnManagementThread.accept() 返回
            └── 发送 "version 3\n" + "hold release\n" + "state on\n" + "bytecount 2\n"
                └── 进入管理接口命令循环
```

**关键特征**：
- **独立进程**：OpenVPN 作为子进程运行，进程崩溃不影响 Java 进程
- **文本协议**：所有交互通过 Unix Domain Socket 的文本行协议
- **FD 传递**：TUN 设备和受保护 Socket 通过 `SCM_RIGHTS` 在 UDS 上传递
- **日志解析**：子进程 stdout 输出带 flags 的日志行，Java 端用正则表达式解析

**配置文件中的管理接口指令**（`VpnProfile.java:407-415`）：

```
management <cacheDir>/mgmtsocket unix
management-client
management-query-passwords
management-hold
```

这些指令让 OpenVPN 2.x 子进程以客户端身份连接 Java 端创建的 UDS 服务端。

#### 11.2.4 OpenVPN 3.x 详细架构

**启动流程**：

```
OpenVPNService.startOpenVPN()
    │
    ├── instantiateOpenVPN3Core()
    │   └── Class.forName("de.blinkt.openvpn.core.OpenVPNThreadv3")
    │       └── 反射实例化（因为 ovpn2 flavor 中不存在此类）
    │
    ├── mManagement = openVPNThreadv3  // 同时作为 Management 接口
    │
    ├── new Thread(openVPNThreadv3, "OpenVPNProcessThread").start()
    │   │
    │   └── OpenVPNThreadv3.run()
    │       ├── setConfig(configString)
    │       │   ├── ClientAPI_Config.setContent(vpnconfig)
    │       │   ├── config.setTunPersist / setGuiVersion / setSsoMethods / ...
    │       │   ├── config.setExternalPkiAlias("extpki")
    │       │   ├── config.setCompressionMode("asym")
    │       │   ├── config.setEnableRouteEmulation(false)  // 使用 App 内部路由模拟
    │       │   └── eval_config(config) → 验证配置
    │       │
    │       ├── setUserPW()
    │       │   └── provide_creds(ClientAPI_ProvideCreds)
    │       │
    │       ├── pollStatus() 定时器（每 2 秒调用 transport_stats()）
    │       │
    │       ├── connect()  ← 阻塞调用，OpenVPN 3.x 核心 JNI
    │       │   │
    │       │   │  [核心运行中，通过虚拟方法回调 Java]
    │       │   │  ├── tun_builder_new()                    → return true
    │       │   │  ├── tun_builder_set_remote_address()     → mService.setMtu()
    │       │   │  ├── tun_builder_set_mtu()                → mService.setMtu()
    │       │   │  ├── tun_builder_add_address()            → mService.setLocalIP/setLocalIPv6
    │       │   │  ├── tun_builder_add_route()              → mService.addRoute
    │       │   │  ├── tun_builder_exclude_route()          → mService.addRoute(exclude)
    │       │   │  ├── tun_builder_reroute_gw()             → mService.addRoute(0.0.0.0/0)
    │       │   │  ├── tun_builder_set_dns_options()        → mService.addDNS/addSearchDomain
    │       │   │  ├── tun_builder_set_proxy_http()         → mService.addHttpProxy
    │       │   │  ├── tun_builder_establish()              → mService.openTun().detachFd()
    │       │   │  ├── socket_protect(socket, remote, ipv6) → mService.protect(socket)
    │       │   │  ├── external_pki_cert_request()          → mVp.getExternalCertificates()
    │       │   │  ├── external_pki_sign_request()          → mVp.getSignedData()
    │       │   │  ├── log(ClientAPI_LogInfo)               → VpnStatus.logInfo()
    │       │   │  ├── event(ClientAPI_Event)               → VpnStatus.updateStateString()
    │       │   │  └── tun_builder_get_local_networks()      → NetworkUtils.getLocalNetworks()
    │       │   │
    │       │   └── connect() 返回 ClientAPI_Status
    │       │
    │       └── 线程结束 → VpnStatus.updateStateString("NOPROCESS")
    │
    └── [无 stdin 写入 — 配置通过 ClientAPI_Config 传递]
```

**关键特征**：
- **进程内线程**：OpenVPN 3.x 核心作为 JNI 库在同一进程内运行
- **虚拟方法回调**：Java 类继承 `ClientAPI_OpenVPNClient`，重写 `tun_builder_*` / `socket_protect` 等方法，C++ 核心通过 SWIG 生成的 JNI 桥接调用
- **直接方法调用**：不需要 UDS 文本协议，不需要 FD 传递（`SCM_RIGHTS`）
- **TUN 建立**：`tun_builder_establish()` 直接返回 `int` FD（`mService.openTun().detachFd()`）
- **Socket 保护**：`socket_protect(int socket, ...)` 直接调用 `mService.protect(socket)`
- **结构化事件**：`event()` 回调提供事件名和信息，无需文本解析
- **主动轮询统计**：`pollStatus()` 定时调用 `transport_stats()` 获取流量数据

**SWIG/JNI 绑定**：

OpenVPN 3.x 核心通过 SWIG 生成 Java 绑定：

```
C++ 核心 (openvpn3/client/ovpncli.hpp)
    │
    │  SWIG 接口定义 (openvpn3/client/ovpncli.i)
    │
    ▼
Java 类 (net.openvpn.ovpn3.*)
    ├── ClientAPI_OpenVPNClient     ← OpenVPNThreadv3 继承此类
    ├── ClientAPI_Config            ← 配置对象
    ├── ClientAPI_EvalConfig        ← 配置验证结果
    ├── ClientAPI_ProvideCreds      ← 认证凭据
    ├── ClientAPI_Status            ← 连接状态
    ├── ClientAPI_Event             ← 事件回调
    ├── ClientAPI_LogInfo           ← 日志回调
    ├── ClientAPI_TransportStats    ← 流量统计
    ├── ClientAPI_ExternalPKICertRequest   ← 外部 PKI 证书请求
    ├── ClientAPI_ExternalPKISignRequest   ← 外部 PKI 签名请求
    ├── DnsOptions / DnsServer / DnsAddress / DnsDomain  ← DNS 配置
    └── ClientAPI_StringVec         ← 字符串数组
```

SWIG 生成任务定义在 `build.gradle.kts:214-248`，编译为 `libovpn3.so`。

#### 11.2.5 OpenVPNManagement 接口的两种实现

`OpenVPNManagement` 接口定义了 VPN 引擎控制方法，两个引擎各自实现：

| 方法 | OpenVpnManagementThread (2.x) | OpenVPNThreadv3 (3.x) |
|------|-------------------------------|----------------------|
| `reconnect()` | `managmentCommand("signal SIGUSR1\n")` + `releaseHold()` | `mHandler.post(() -> reconnect(1))` JNI 调用 |
| `pause(reason)` | `signalusr1()` — 发送 SIGUSR1 | `mHandler.post(() -> super.pause(reason.toString()))` JNI 调用 |
| `resume()` | `releaseHold()` — 发送 `hold release\n` | `releaseHold()` — 但 v3 中是空操作 |
| `stopVPN(replace)` | `managmentCommand("signal SIGINT\n")` | `mHandler.post(this::stop)` JNI 调用 |
| `networkChange(same)` | `managmentCommand("network-change...")` | `mHandler.post(() -> reconnect(1))` |
| `sendCRResponse(resp)` | `managmentCommand("cr-response " + resp + "\n")` | `mHandler.post(() -> post_cc_msg("CR_RESPONSE," + resp))` |
| `setPauseCallback(cb)` | 设置回调用于判断是否应恢复运行 | 空操作 |

OpenVPNThreadv3 还额外实现了 `OpenVPNManagement` 中未定义的回调方法（由 `ClientAPI_OpenVPNClient` 声明）：

| 回调方法 | 触发时机 | 作用 |
|----------|----------|------|
| `tun_builder_new()` | TUN 设备创建开始 | 初始化（返回 true） |
| `tun_builder_set_mtu(int)` | 设置 MTU | `mService.setMtu()` |
| `tun_builder_set_remote_address()` | 设置远端地址 | `mService.setMtu()` |
| `tun_builder_add_address()` | 设置本地 IP | `mService.setLocalIP/setLocalIPv6` |
| `tun_builder_add_route()` | 添加路由 | `mService.addRoute()` |
| `tun_builder_exclude_route()` | 排除路由 | `mService.addRoute(exclude=false)` |
| `tun_builder_reroute_gw()` | 重定向默认网关 | `mService.addRoute("0.0.0.0/0")` |
| `tun_builder_set_dns_options()` | 设置 DNS | `mService.addDNS/addSearchDomain` |
| `tun_builder_set_proxy_http()` | 设置 HTTP 代理 | `mService.addHttpProxy()` |
| `tun_builder_establish()` | 建立 TUN 设备 | `mService.openTun().detachFd()` → 返回 int FD |
| `tun_builder_set_session_name()` | 设置会话名 | 日志记录 |
| `tun_builder_set_layer(int)` | 设置网络层 | 检查 layer == 3（仅支持 L3/TUN） |
| `tun_builder_get_local_networks()` | 获取本地网络 | `NetworkUtils.getLocalNetworks()` |
| `socket_protect(int, String, boolean)` | 保护 Socket | `mService.protect(socket)` |
| `external_pki_cert_request()` | 外部 PKI 证书请求 | `mVp.getExternalCertificates()` |
| `external_pki_sign_request()` | 外部 PKI 签名请求 | `mVp.getSignedData()`（使用 SignaturePadding 枚举） |
| `log(ClientAPI_LogInfo)` | 日志输出 | `VpnStatus.logInfo()` |
| `event(ClientAPI_Event)` | 状态事件 | `VpnStatus.updateStateString()` |
| `pause_on_connection_timeout()` | 连接超时暂停 | 返回 true（暂停而非断开） |

#### 11.2.6 OpenVPN 3.x 的限制

`VpnProfile.checkProfile()` 中检查 OpenVPN 3.x 不支持的配置（`VpnProfile.java:1064-1074`）：

```java
if (useOpenVPN3) {
    if (mAuthenticationType == TYPE_STATICKEYS)
        return R.string.openvpn3_nostatickeys;    // 不支持静态密钥
    if (mAuthenticationType == TYPE_PKCS12 || mAuthenticationType == TYPE_USERPASS_PKCS12)
        return R.string.openvpn3_pkcs12;           // 不支持 PKCS#12 证书
    for (Connection conn : mConnections) {
        if (conn.mProxyType == Connection.ProxyType.ORBOT || conn.mProxyType == Connection.ProxyType.SOCKS5)
            return R.string.openvpn3_socksproxy;   // 不支持 SOCKS5 代理
    }
}
```

| 不支持的功能 | 原因 |
|-------------|------|
| 静态密钥认证（`TYPE_STATICKEYS`） | OpenVPN 3.x 核心不实现静态密钥模式 |
| PKCS#12 证书（`TYPE_PKCS12`） | OpenVPN 3.x 使用外部 PKI 接口，不支持 PKCS#12 格式 |
| SOCKS5 代理 / Orbot 代理 | OpenVPN 3.x 核心不内置 SOCKS5 代理支持 |

#### 11.2.7 完整对比图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        OpenVPNService.startOpenVPN()                            │
│                                                                                 │
│   useOpenVPN3 = VpnProfile.doUseOpenVPN3(context)                              │
│               = SharedPreferences("ovpn3") && BuildConfig.openvpn3             │
│                                                                                 │
│   ┌───────────────────────────┬─────────────────────────────────────────────┐   │
│   │    useOpenVPN3 == false   │           useOpenVPN3 == true               │   │
│   │    (OpenVPN 2.x 路径)     │           (OpenVPN 3.x 路径)                 │   │
│   ├───────────────────────────┼─────────────────────────────────────────────┤   │
│   │                           │                                             │   │
│   │  OpenVpnManagementThread  │  instantiateOpenVPN3Core()                  │   │
│   │  ├── 创建 UDS 服务端      │  ├── 反射加载 OpenVPNThreadv3               │   │
│   │  ├── bind(mgmtsocket)     │  └── mManagement = openVPNThreadv3          │   │
│   │  ├── listen()             │                                             │   │
│   │  └── accept() [等待连接]  │  无 UDS — 不需要管理接口线程                  │   │
│   │                           │                                             │   │
│   │  mManagement = 管理线程   │  mManagement = openVPNThreadv3              │   │
│   │                           │  (同时实现 OpenVPNManagement 接口)           │   │
│   │  OpenVPNThread            │  OpenVPNThreadv3                            │   │
│   │  ├── ProcessBuilder       │  ├── System.loadLibrary("ovpn3")            │   │
│   │  ├── 启动 libovpnexec.so  │  ├── extends ClientAPI_OpenVPNClient        │   │
│   │  ├── 子进程独立运行        │  ├── 进程内线程运行                         │   │
│   │  ├── stdout 日志正则解析   │  ├── log() 结构化回调                       │   │
│   │  └── stdin 写入配置       │  └── ClientAPI_Config.setContent()          │   │
│   │                           │                                             │   │
│   │  ┌──── 通信方式 ────┐     │  ┌──── 通信方式 ────┐                      │   │
│   │  │ Unix Domain Socket│    │  │ Java 虚拟方法回调 │                      │   │
│   │  │ 文本行协议        │    │  │ JNI 直接调用      │                      │   │
│   │  │                   │    │  │                   │                      │   │
│   │  │ TUN 建立:         │    │  │ TUN 建立:         │                      │   │
│   │  │  >NEED-OK:OPENTUN │    │  │  tun_builder_     │                      │   │
│   │  │  → openTun()      │    │  │  establish()      │                      │   │
│   │  │  → SCM_RIGHTS 传FD│    │  │  → openTun()      │                      │   │
│   │  │                   │    │  │  → return int fd   │                      │   │
│   │  │ Socket 保护:      │    │  │ Socket 保护:       │                      │   │
│   │  │  PROTECTFD:       │    │  │  socket_protect()  │                      │   │
│   │  │  → SCM_RIGHTS 传FD│    │  │  → protect(socket) │                      │   │
│   │  │  → protect(fd)    │    │  │                   │                      │   │
│   │  │ 路由/DNS:         │    │  │ 路由/DNS:         │                      │   │
│   │  │  >NEED-OK:ROUTE   │    │  │  tun_builder_add_  │                      │   │
│   │  │  >NEED-OK:IFCONFIG│    │  │  route()           │                      │   │
│   │  │  >NEED-OK:DNSSERVER│   │  │  tun_builder_set_  │                      │   │
│   │  │                   │    │  │  dns_options()     │                      │   │
│   │  │ 签名请求:         │    │  │ 签名请求:         │                      │   │
│   │  │  >PK_SIGN:data,..│    │  │  external_pki_     │                      │   │
│   │  │  → 文本解析参数   │    │  │  sign_request()    │                      │   │
│   │  │                   │    │  │  → 结构化参数       │                      │   │
│   │  │ 状态/日志:        │    │  │ 状态/日志:         │                      │   │
│   │  │  >STATE:...       │    │  │  event() 回调      │                      │   │
│   │  │  >LOG:...         │    │  │  log() 回调         │                      │   │
│   │  │  >BYTECOUNT:...   │    │  │  pollStatus() 轮询  │                      │   │
│   │  │                   │    │  │                   │                      │   │
│   │  │ 控制:             │    │  │ 控制:              │                      │   │
│   │  │  signal SIGINT    │    │  │  stop() JNI        │                      │   │
│   │  │  signal SIGUSR1   │    │  │  reconnect() JNI   │                      │   │
│   │  │  hold release     │    │  │  pause() JNI       │                      │   │
│   │  │  network-change   │    │  │  reconnect(1) JNI  │                      │   │
│   │  └───────────────────┘    │  └───────────────────┘                      │   │
│   │                           │                                             │   │
│   │  支持所有认证方式         │  不支持: 静态密钥, PKCS#12, SOCKS5            │   │
│   │  进程崩溃不影响 Java      │  进程崩溃会带崩 Java                         │   │
│   │  需要管理接口协议解析     │  结构化 API，无需文本解析                     │   │
│   │  需要 FD 传递 (SCM_RIGHTS)│  直接返回 int fd，无需 SCM_RIGHTS            │   │
│   └───────────────────────────┴─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 11.2.8 关键源文件索引

| 文件 | 行号 | 内容 |
|------|------|------|
| `VpnProfile.java` | 227-233 | `doUseOpenVPN3()` — useOpenVPN3 决策（偏好 + BuildConfig） |
| `VpnProfile.java` | 1023-1075 | `checkProfile()` — OpenVPN 3.x 不支持的配置检查 |
| `OpenVPNService.java` | 701-740 | `startOpenVPN()` — 2.x/3.x 分支启动逻辑 |
| `OpenVPNService.java` | 786-795 | `instantiateOpenVPN3Core()` — 反射实例化 OpenVPNThreadv3 |
| `OpenVPNThread.java` | 34-210 | OpenVPN 2.x 进程线程：ProcessBuilder 启动子进程、日志解析 |
| `OpenVPNThreadv3.java` | 35-429 | OpenVPN 3.x 线程：继承 ClientAPI_OpenVPNClient，虚拟方法回调 |
| `OpenVPNThreadv3.java` | 57-75 | `run()` — setConfig + connect() 主循环 |
| `OpenVPNThreadv3.java` | 177-179 | `tun_builder_establish()` — TUN FD 返回 |
| `OpenVPNThreadv3.java` | 314-316 | `socket_protect()` — Socket 保护 |
| `OpenVPNThreadv3.java` | 220-259 | `setConfig()` — ClientAPI_Config 配置 |
| `OpenVPNThreadv3.java` | 353-379 | `event()` — 状态事件回调 |
| `OpenVPNThreadv3.java` | 343-350 | `log()` — 日志回调 |
| `OpenVPNThreadv3.java` | 423-428 | `pollStatus()` — 流量统计轮询 |
| `OpenVpnManagementThread.java` | 117-800 | OpenVPN 2.x 管理接口（UDS 文本协议） |
| `VPNLaunchHelper.java` | 50-63 | `buildOpenvpnArgv()` — 构建 OpenVPN 2.x 命令行参数 |
| `NativeUtils.java` | 63-70 | 原生库加载：`ovpnutil`, `osslspeedtest` |
| `NativeUtils.java` | 29-31 | `getOpenVPN2GitVersion()` / `getOpenVPN3GitVersion()` — 版本查询 |
| `AboutFragment.java` | 92-96 | About 页面根据 BuildConfig.openvpn3 显示版本信息 |
| `build.gradle.kts` | 115-136 | Product Flavors：ovpn23 (openvpn3=true) / ovpn2 (openvpn3=false) |
| `build.gradle.kts` | 214-248 | SWIG 代码生成任务：ovpncli.i → net.openvpn.ovpn3.* Java 类 |
| `cpp/openvpn3/client/ovpncli.i` | — | SWIG 接口定义文件 |
| `cpp/openvpn3/` | — | OpenVPN 3.x C++ 核心源码 |
