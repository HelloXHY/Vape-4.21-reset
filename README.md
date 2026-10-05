# Vape 4 Product Recovery

Vape 4 Java 层与 Windows x64 原生桥接层的研究性恢复工程。

### 对 Vape 官方源码进行还原 原始发布包或厂商签名产物，不保证具备与原产品完全一致的行为

> 本项目用于软件恢复、兼容性分析和自有环境测试。仅应在你拥有并获准测试的隔离实例中
> 使用，并自行确认当地法律、软件许可和服务器规则。

## 当前状态

| 范围      | 状态                                             |
| ------- | ---------------------------------------------- |
| Java 源码 | 2,939 个样本自有包源码，可由 Gradle 正常编译                  |
| 资源      | 242 个映射、字体、纹理、着色器、声音及本地化资源                     |
| 注入载荷    | 自包含 Shadow JAR，目标为 Java 8 class-file major 52  |
| 原生桥接    | Windows x64 JNI/JVMTI DLL 与 `LoadLibraryW` 注入器 |
| 运行验证    | 构建与载荷结构已验证                                     |

当前支持 **Minecraft 1.7.10 Forge、1.7.10 Vanilla、1.8.9 Forge、1.8.9 Vanilla、1.12.2 Forge、1.12.2 Vanilla、1.21.11 Forge、1.21.11 Vanilla、26.2 Forge 和 26.2 Vanilla**；
其中 1.21.11 Forge 已在 Forge 61.0.8 上验证，26.2 Forge 已在 Forge 65.1.0 上验证。也支持向启用了 Forge 的 Lunar Client 实例注入。
Fabric 不受支持。Minecraft 1.16.5 的支持不佳，部分映射、渲染和模块功能可能
无法正常工作。所有目标实例均须使用 64 位 JVM。

## 环境要求

仅编译和校验 Java 层需要：

- JDK 17，用作 Gradle toolchain；输出默认通过 `--release 8` 编译
- 项目自带的 Gradle Wrapper；构建脚本固定要求 Gradle 8.8
- 可访问 Maven Central 和 Gradle Plugin Portal 的网络连接

构建 native bundle 还需要：

- Windows x64
- Visual Studio 2022 C++ x64 工具链及 Windows SDK
- CMake 3.21 或更高版本
- 一套包含 JNI/JVMTI 头文件的 JDK；面向 1.7.10、1.8.9 和 1.12.2 测试时建议使用 JDK 8

## 快速开始

在 PowerShell 中进入仓库根目录：

```powershell
.\gradlew.bat clean build verifyInjectionPayload
```

该命令会完成以下工作：

1. 编译恢复源码并处理全部资源。
2. 检查源码数量以及残留的致命 CFR 反编译标记。
3. 生成包含运行时依赖的 injection JAR。
4. 确认载荷包含必要包，且所有 class 均可由 Java 8 加载。

主要 Java 产物位于 `build/libs/`。如需生成 IntelliJ IDEA 工程配置，可运行：

```powershell
.\gradlew.bat idea
```

## 构建原生测试包

```powershell
.\gradlew.bat prepareInjectionBundle -PtargetRelease=8 `
  -PnativeJavaHome="C:\Program Files\Java\latest\jre-1.8"
```

完整测试包输出到 `build/injection/`：

```text
Vape421Native.dll
Vape421Injector.exe
README.md
```

DLL 将 Java injection JAR 作为 `RCDATA` 嵌入，不要求另行放置 payload。原生桥接层只实现
从样本九项 `RegisterNatives` 表恢复出的接口；未在样本中注册的额外 Java native 声明
不会被臆造实现。更多细节见 [`native/README.md`](native/README.md)。

## 隔离环境运行

启动使用 64 位 JVM 的受支持 Minecraft 实例，或启用了 Forge 的 Lunar Client 实例后，在
`build/injection/` 中执行：

```powershell
.\Vape421Injector.exe <pid> .\Vape421Native.dll
```

注入器仅执行 `LoadLibraryW`。DLL 加载后会等待 JVM 与 Minecraft `Client thread`，通过其
上下文 ClassLoader 加载内嵌 JAR，注册九个 native 方法，并调用
`gg.vape.runtime.NativeBridge.start()`。执行结果写入 DLL 同目录的
`vape421-native.log`。

## 常用校验任务

| 命令                                     | 用途                      |
| -------------------------------------- | ----------------------- |
| `.\gradlew.bat check`                  | 编译、源码覆盖与恢复质量检查          |
| `.\gradlew.bat injectionJar`           | 构建自包含 Java 注入载荷         |
| `.\gradlew.bat verifyInjectionPayload` | 检查依赖完整性与 Java 8 字节码版本   |
| `.\gradlew.bat buildNative`            | 构建 x64 DLL 和注入器         |
| `.\gradlew.bat prepareInjectionBundle` | 汇总可供隔离测试的 native bundle |
