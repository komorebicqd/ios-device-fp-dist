# DeviceFingerprintSDK 商户接入文档

## 目录

1. [产品简介](#产品简介)
2. [交付方式](#交付方式)
3. [环境要求](#环境要求)
4. [接入步骤](#接入步骤)
5. [代码示例](#代码示例)
6. [配置参数说明](#配置参数说明)
7. [接口语义](#接口语义)
8. [错误处理](#错误处理)
9. [二进制完整性校验](#二进制完整性校验)
10. [常见问题](#常见问题)
11. [合规说明](#合规说明)
12. [Demo App](#demo-app)

---

## 产品简介

DeviceFingerprintSDK 是一款面向 iOS 应用的设备指纹 SDK，用于在服务端稳定识别一台物理设备。

- **合规**：不使用 Apple 私有 API、不越狱检测、不采集 IDFA。所有采集字段均属于 iOS 公开信息。
- **零依赖**：仅使用 `Foundation`、`Security`、`CryptoKit`、`UIKit`，不引入任何第三方库。
- **离线交付**：以 XCFramework 二进制形式交付，商户无需源码仓库，也不需要 Swift Package Manager。
- **高可靠**：冷启动并发调用自动单飞、三级缓存（内存 → Keychain → 网络）、服务端时间校准、完整加密签名链路。

---

## 交付方式

商户通过公开分发仓库下载离线交付包：

- **仓库**：`https://github.com/komorebicqd/ios-device-fp-dist`
- **下载链接**：`https://raw.githubusercontent.com/komorebicqd/ios-device-fp-dist/main/DeviceFingerprintSDK-Offline.zip`
- **ZIP SHA-256**：`674f1ad831983a69dcedb2625278dce727598c1657cd8205189a00328d0aa6c2`

ZIP 内包含：

| 文件 | 说明 |
| --- | --- |
| `DeviceFingerprintSDK.xcframework` | 已编译的 SDK 二进制，含真机 arm64 与模拟器 arm64/x86_64 |
| `INTEGRATION.md` | 精简版接入说明 |
| `SHA256SUMS.txt` | 真机与模拟器二进制的 SHA-256 校验值 |

---

## 环境要求

| 项目 | 要求 |
| --- | --- |
| 最低 iOS 版本 | 13.0 |
| 建议 Xcode 版本 | 16.4 或更高 |
| 真机架构 | arm64 |
| 模拟器架构 | arm64（Apple Silicon）、x86_64（Intel） |
| Swift 版本 | Swift 5 模式（SDK 使用 Swift 6.1.2 编译） |

---

## 接入步骤

### 第一步：解压

将 `DeviceFingerprintSDK-Offline.zip` 解压到任意位置，得到 `DeviceFingerprintSDK.xcframework`。

### 第二步：添加到 Xcode 工程

1. 使用 Xcode 打开商户 App 工程。
2. 在项目导航器中选择 **App Target**。
3. 进入 **General** 标签页，找到 **Frameworks, Libraries, and Embedded Content**。
4. 点击列表下方的 `+` 按钮。
5. 在弹出窗口中选择 **Add Other → Add Files…**。
6. 选择解压后的 `DeviceFingerprintSDK.xcframework`。
7. 添加后，将其 Embed 设置为 **Embed & Sign**。

> **注意**：请添加最外层的 `.xcframework`，不要单独添加某个架构目录下的 `.framework`。Xcode 会根据当前构建目标自动选择正确的切片。

### 第三步：代码调用

在 App 启动阶段（例如 `AppDelegate.didFinishLaunching` 或 `App` 的 `body` 中）调用 `configure`，随后在任意位置调用 `getDeviceId`。具体代码见下一节。

---

## 代码示例

### 回调方式（推荐用于兼容 iOS 13）

```swift
import UIKit
import DeviceFingerprintSDK

@main
class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        // 只需调用一次
        FingerprintSDK.configure(
            appId: "your-app-id",
            baseURL: "https://api.example.com"
        )

        // 获取 deviceId（回调不保证在主线程）
        FingerprintSDK.getDeviceId { result in
            DispatchQueue.main.async {
                switch result {
                case .success(let deviceId):
                    print("deviceId: \(deviceId)")
                case .failure(let error):
                    print("获取失败: \(error.localizedDescription)")
                }
            }
        }

        return true
    }
}
```

### async/await 方式

```swift
import DeviceFingerprintSDK

Task {
    do {
        let deviceId = try await FingerprintSDK.getDeviceId()
        print("deviceId: \(deviceId)")
    } catch {
        print("获取失败: \(error.localizedDescription)")
    }
}
```

### SwiftUI 示例

```swift
import SwiftUI
import DeviceFingerprintSDK

@main
struct MyApp: App {
    init() {
        FingerprintSDK.configure(
            appId: "your-app-id",
            baseURL: "https://api.example.com"
        )
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

struct ContentView: View {
    @State private var deviceId: String = "获取中..."

    var body: some View {
        Text(deviceId)
            .task {
                do {
                    deviceId = try await FingerprintSDK.getDeviceId()
                } catch {
                    deviceId = "获取失败: \(error.localizedDescription)"
                }
            }
    }
}
```

### 切换配置

同一 App 进程内切换 `appId` 或 `baseURL` 前，需要先调用 `reset()` 清空内部缓存：

```swift
FingerprintSDK.reset()
FingerprintSDK.configure(appId: "another-app-id", baseURL: "https://another.example.com")
```

`reset()` 只清空内存状态，**不会删除** Keychain 中已持久化的 `deviceId` 和 `clientAnchorId`。

---

## 配置参数说明

### `appId`

服务方分配的商户标识字符串，在 `/init` 请求中作为请求体的一部分发送。**同一商户下的不同 App 会使用不同的 appId**。

### `baseURL`

指纹服务的 HTTPS 地址。例如 `https://api.example.com`。SDK 会依次请求 `POST /fingerprint/init` 和 `POST /fingerprint/generate`。

要求：

- 必须以 `http://` 或 `https://` 开头
- 生产环境强烈建议使用 `https://`
- 不得包含路径后缀（例如不要写 `https://api.example.com/v1`）

---

## 接口语义

### 三级缓存

`getDeviceId` 按以下优先级返回：

1. **内存缓存** — 同进程内已获取过的 deviceId，直接同步返回。
2. **Keychain 缓存** — App 上次启动时已持久化的 deviceId，从 Keychain 读取后返回。Keychain 使用 `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` 权限，**卸载重装后仍可保留**。
3. **网络链路** — 前两级均未命中时，发起完整的 `/init → /generate` 请求链路。

### 并发单飞

同一时刻多个 `getDeviceId` 调用只触发一次 `/init → /generate` 链路，其余调用挂起并共享同一次的结果。这保证：

- 冷启动时即使 UI、网络层、业务逻辑同时触发，也只会产生一次网络请求
- 所有等待者收到完全相同的 `deviceId`
- 链路失败时所有等待者收到同一错误，并允许后续调用重新发起（**不缓存失败**）

### 服务端时间校准

SDK 在 `/init` 响应中读取 `serverTimestamp`，并计算与本地时间的偏移量。后续所有请求的时间戳（包括签名计算中的 `timestamp` 字段）均使用校准后的毫秒时间戳，避免设备本地时钟漂移导致签名被拒。

### Keychain 中的持久化字段

| Key | 写入时机 | 用途 |
| --- | --- | --- |
| `clientAnchorId` | 首次调用 `/generate` 前 | 36 位 UUID，卸载重装后保持不变，用作设备的稳定锚点 |
| `deviceId` | `/generate` 成功后 | 服务端返回的设备指纹 ID，直接作为缓存命中 |

### /init → /generate 链路概览

```
客户端                         服务端
  │                              │
  │── POST /fingerprint/init ───>│
  │   appId / bundleId / tempId  │
  │   timestamp                  │
  │                              │
  │<── nonce / sessionToken      │
  │    rsaPublicKey              │
  │    serverTimestamp           │
  │                              │
  │  (采集 14 项原始设备信息)      │
  │  (生成 clientAnchorId)        │
  │  (AES-256-GCM 加密原始数据)    │
  │  (RSA-OAEP-SHA256 加密 AES 密钥)│
  │  (HMAC-SHA256 签名请求串)      │
  │                              │
  │── POST /fingerprint/generate >│
  │   加密数据 + 签名 + 时间戳    │
  │                              │
  │<── { deviceId, success } ────│
  │                              │
  │  (写入 Keychain 持久化)       │
```

---

## 错误处理

`getDeviceId` 的回调或 `throws` 可能返回以下错误（`FingerprintError`）：

| 错误 | 含义 | 建议 |
| --- | --- | --- |
| `notConfigured` | 未调用 `configure` 就调用了 `getDeviceId` | 确保 App 启动时先调用 `configure` |
| `invalidBaseURL(String)` | `baseURL` 不是有效的 http/https URL | 检查 `baseURL` 格式 |
| `encoding(String)` | 请求体 JSON 编码失败 | 内部错误，通常不需要处理 |
| `invalidResponse(String)` | 服务端响应解析失败或字段不完整 | 检查服务端实现或联系服务方 |
| `serverRejected(String)` | 服务端拒绝了请求（如签名校验失败、nonce 已使用） | 检查 appId、baseURL、RSA 公钥是否匹配当前环境 |
| `keychainFailed(OSStatus)` | Keychain 读写失败 | 多见于模拟器或签名环境异常；真机上通常不会出现 |
| `encryptionFailed(String)` | AES/RSA/HMAC 加密或签名失败 | 内部错误，通常不需要处理 |
| `networkError(Error)` | 网络层错误（超时、无连接等） | 建议上层在合适时机重试 |

### 重试建议

- **网络错误**（`networkError`）：指数退避重试 2~3 次即可；服务端也需要按 nonce 和 sessionToken 正确处理重放请求。
- **服务端拒绝**（`serverRejected`）：不要盲目重试，先排查配置问题。
- **其他错误**：通常不需要重试，交由服务方排查。

---

## 二进制完整性校验

在 macOS 终端，进入解压后的 ZIP 目录，执行：

```bash
shasum -a 256 -c SHA256SUMS.txt
```

两行均显示 `OK` 即表示二进制文件完整。

交付包整体校验：

```bash
shasum -a 256 DeviceFingerprintSDK-Offline.zip
# 预期：674f1ad831983a69dcedb2625278dce727598c1657cd8205189a00328d0aa6c2
```

---

## 常见问题

### Q: 添加后构建报错 `No such module 'DeviceFingerprintSDK'`

检查 `.xcframework` 是否正确添加到 App Target 的 **Frameworks, Libraries, and Embedded Content**，确认设置不为空，然后执行 **Product → Clean Build Folder**（⇧⌘K）后重新构建。

### Q: 真机上链接错误，但模拟器正常

确认添加的是最外层 `.xcframework`，不是 `ios-arm64_x86_64-simulator` 目录下的 `.framework`。真机和模拟器是同一 `.xcframework` 中的两个切片，Xcode 会自动选择。

### Q: 冷启动时 UI 卡顿

SDK 的采集、加密签名和 `/init → /generate` 请求均在非主线程执行，`getDeviceId` 的回调也不保证在主线程。如果业务需要在主线程使用 deviceId，请自行切换队列：

```swift
DispatchQueue.main.async {
    // 使用 deviceId 更新 UI
}
```

### Q: 同一台设备卸载重装后 deviceId 会变吗？

**不会**。SDK 使用 Keychain 持久化 `clientAnchorId` 和 `deviceId`，权限为 `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`。卸载重装后，Keychain 条目仍然存在，`deviceId` 会直接从 Keychain 缓存返回。

如果需要彻底重置（测试用），调用 `reset()` 只会清空内存，Keychain 中的值仍保留。要删除 Keychain 中的条目，请在真机上通过 Xcode 的 **Devices and Simulators → 卸载 App**（不会删 Keychain），或者通过系统设置 → 还原所有设置。

### Q: 本地 HTTP 联调时真机无法访问

真机访问 `http://192.168.x.x:端口` 之类的局域网服务时，需要在 App 的 Info.plist 中添加 `NSLocalNetworkUsageDescription` 键值。该设置仅用于开发联调，生产环境请使用 HTTPS。

### Q: 可以只用 Swift Package Manager 接入吗？

可以，但**不推荐**。如果商户工程使用 SPM 并希望完全离线，可以将 XCFramework ZIP 放到本地或私有制品库，然后在 Package.swift 中通过本地路径或 file URL 声明 `binaryTarget`。但这种方式不如 Xcode 直接添加 `.xcframework` 直观。

---

## 合规说明

DeviceFingerprintSDK 严格遵循以下原则：

- **不使用私有 API**：所有采集字段均来自 `UIDevice`、`UIScreen`、`uname`、`ProcessInfo`、`Locale`、`TimeZone` 等公开系统接口。
- **不采集 IDFA**：SDK 不读取 `ASIdentifierManager`。
- **不使用越狱检测或黑产手法**：不调用 `fork`、`dyld`、`CFRunLoop` 之外的私有符号、不检查 `/Applications/Cydia.app`、不尝试 `dlopen("/usr/lib/libjailbreak.dylib")`。
- **所有二进制位于 XCFramework 内**：不执行远程脚本、不包含热更新能力。

当前采集的 14 项原始设备信息：

| 字段 | 来源 |
| --- | --- |
| `clientAnchorId` | SDK 首次运行生成的 UUID，写入 Keychain |
| `idfv` | `UIDevice.current.identifierForVendor?.uuidString` |
| `deviceModel` | `uname.machine`（例如 `iPhone16,2`） |
| `osVersion` | `UIDevice.current.systemVersion` |
| `screenWidth` | `UIScreen.main.bounds.width` 整数像素 |
| `screenHeight` | `UIScreen.main.bounds.height` 整数像素 |
| `screenScale` | `UIScreen.main.scale` |
| `physicalMemory` | `ProcessInfo.processInfo.physicalMemory` |
| `totalDiskSpace` | 文件系统总容量（`attributesOfFileSystem(forPath: "/")`） |
| `cpuCount` | `ProcessInfo.processInfo.processorCount` |
| `systemLanguages` | `Locale.preferredLanguages` |
| `regionCode` | `(Locale.current as NSLocale).object(forKey: .countryCode)` |
| `timeZone` | `TimeZone.current.identifier` |
| `isSimulator` | 编译期 `#if targetEnvironment(simulator)` |

如果 App 计划上架 App Store，请注意：

- `totalDiskSpace` 字段涉及「Disk Space Required Reason」API，需要在 `Info.plist` 中声明 `NSUserTrackingUsageDescription` 或改用 Disk APIs 的等价理由声明。SDK 本身不弹出任何系统权限弹窗，但商户 App 需要自行处理合规声明。
- 当前版本未附带 `PrivacyInfo.xcprivacy`，商户上架时可将 SDK 采集字段填写进自己的 `Privacy Manifest`。

---

## Demo App

源码仓库的 `Demo/FingerprintDemo` 是一个最小 SwiftUI 工程，用于真机联调。它通过本地相对路径依赖 SDK 源码包，包含：

- `appId` / `baseURL` 可编辑的配置面板
- 单次获取 deviceId
- 并发 10 次获取（验证单飞语义）
- 重置内存缓存
- 日志面板显示完整链路输出

Demo App 仅供开发参考，**不在离线交付包中**。商户如需要 Demo，请另行索取。
