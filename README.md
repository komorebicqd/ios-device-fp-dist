# DeviceFingerprintSDK

设备指纹 SDK 的二进制分发仓库（SwiftPM binaryTarget）。仓库内只有 SwiftPM 清单与 XCFramework 产物，不含源码。

## 要求

- iOS 13.0+
- Xcode 14+

## 接入

Xcode -> File -> Add Package Dependencies...，填入：

```
https://github.com/komorebicqd/ios-device-fp-dist.git
```

版本规则选择 Up to Next Major from 1.0.0。或在 Package.swift 中：

```swift
.package(url: "https://github.com/komorebicqd/ios-device-fp-dist.git", from: "1.0.0")
```

## 使用

```swift
import DeviceFingerprintSDK

FingerprintSDK.configure(appId: "<服务方分配的 appId>", baseURL: "https://api.example.com")

FingerprintSDK.getDeviceId { result in
    switch result {
    case .success(let deviceId): print(deviceId)
    case .failure(let error): print(error)
    }
}

// 或 async/await
let deviceId = try await FingerprintSDK.getDeviceId()
```

`configure` 建议在 App 启动时调用一次；`getDeviceId` 可在任意线程、任意次数调用，SDK 内部会合并并发请求并缓存结果。

## 版本

版本号对应上游发布标签（v1.0.0 -> 1.0.0）。升级时在 Xcode 执行 Update to Latest Package Versions。
