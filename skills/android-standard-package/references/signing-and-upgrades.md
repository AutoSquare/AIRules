# 签名、构建与覆盖升级

## 配置与 Android Studio 向导

先确认正在为哪个项目打包。新项目由项目根目录保管各自的 `<项目>-release.jks` 和 `key.properties`；为两者添加忽略规则，用 `git check-ignore` 核对，同时检查它们从未被跟踪。现有项目沿用已发布的发行私钥和约定位置。由密钥持有人另外保管可恢复的安全备份。示例、日志、回复、截图和提交均不能包含密码或私钥。

`key.properties` 的四个键是 `storeFile`、`storePassword`、`keyAlias`、`keyPassword`。构建读取本机文件。布局约定为根目录时，让 Gradle 相对于 **rootProject** 解析 storeFile，避免模块目录和 Windows 反斜杠转义导致定位错误。

| 向导字段 | 填写来源 |
| --- | --- |
| Module / 类型 | 确认是要发行的应用模块；侧载选择 APK，构建变体选择 release。 |
| Key store path | 选择本项目已有发行库。首次发版且确认无历史身份时才新建。 |
| Key store password | 由用户在本机填写 storePassword。 |
| Key alias | 使用已核实的 keyAlias；可通过向导选择已有私钥条目。 |
| Key password | 由用户在本机填写 keyPassword。两个密码可能相同，也可能不同。 |

`Cannot recover key` 时，按“当前项目和库路径 → 库能否用库密码打开 → alias 是否是正确的 PrivateKeyEntry → 私钥密码是否正确”逐项缩小范围。可用 `keytool -list -v -keystore <库路径> -alias <别名>`，由用户在本机交互输入密码。能列出证书仅证明能读取条目，不证明能解锁私钥。按实际配置重新签一个本地 release 验证。不要把另一项目的密码填进来，也不要为绕过错误覆盖旧库。

## 产物与附件

1. 使用项目约定的 release 构建命令，例如 Windows 上的 `.\gradlew.bat :app:assembleRelease`。确认发行变体指向正式环境，不把 localhost、测试包后缀或调试签名带入附件。
2. 新项目从 versionName 生成 `<产品名>-v<versionName>-release.apk`。若向导仍给出 `app-release.apk`，查当前 Android Gradle Plugin 的产物配置及向导实际输出目录，同时核对命令行和向导的结果。已有元数据或基线配置引用文件名时要保持一致。
3. 使用 Android SDK 工具读取**最终交付的文件**，工具由本机 SDK 定位。`apkanalyzer manifest application-id <APK>`、`manifest version-name`、`manifest version-code`、`manifest min-sdk` 分别读取身份与版本。用 `apksigner verify --verbose --print-certs <APK>` 验证签名及签名证书 SHA-256。与上一正式版核对 applicationId、签名连续性及递增的 versionCode。
4. 用文件 SHA-256 比对核验过的 APK 与待上传附件。它标识某个文件，和用于判断发行身份的**证书指纹**用途不同。仅向下载者提供所需安装包；本地密钥、密码文件、构建元数据和基线分析文件均无需作为侧载附件上传。

## 覆盖安装不兼容

先查所选设备上的包名与版本。需要比对证书时，用 `adb -s <序列号> shell pm path <包名>` 找到 base APK，将 APK 复制到临时目录后用同一 apksigner 读取指纹。仅处理安装包，无须读取应用私有数据。

- 同包名、同发行身份且版本兼容：在被指定设备上执行覆盖安装，核对安装后版本、已有登录与关键功能。
- `INSTALL_FAILED_UPDATE_INCOMPATIBLE` 且确认签名不同：停止覆盖尝试，保留原应用和登录数据。可用同源码同配置、与已装包同签名的既有测试变体验证生产功能，或在另行指定的干净设备测试正式 APK。清数据、卸载或更换发行密钥不是常规重试步骤。
- 发生版本降级或设备低于 minSdk：记录具体条件，改用兼容版本或合适设备；勿把两者归因于签名。

**实际判例。** WePass 第一个面向公众的正式版是 v1.0.5。网站和客户端共用一个远端，发布时应选择含 APK 源码的 AndroidShell 分支提交。模拟器的旧包来自测试证书，正式 APK 覆盖被拒；同签名测试构建成功连接正式站，验证了外观功能。记录应分别写清正式 APK 的验签结果和测试构建的设备实测结果。
