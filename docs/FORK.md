# comic-komelia（Komelia fork）

安卓漫画阅读端，服务这套自建漫画栈：

| 组件 | 仓库 | 职责 |
|---|---|---|
| 下载 | `comic-mangrab` | 搜索 / 下载 / 打包 CBZ / 落 115 |
| 元数据 | `comic-database`（comic_meta） | PostgreSQL 元数据与归档状态权威库 |
| 服务端 | `comic-komga` | Komga 1.30.1 fork：浏览 / 阅读 / 进度 |
| **阅读端** | **`comic-komelia`（本仓库）** | 安卓 App，连 Komga 读漫画 |

- 上游：https://github.com/Snd-R/Komelia （fork 自 `0.20.0`）
- 本 fork 只做「让它在我们的部署上能正常工作」的必要改动，尽量不碰上游代码结构，
  方便随时 `git merge upstream/main`。

## 与这套栈的对接

| 项 | 值 |
|---|---|
| 服务器地址 | `http://192.168.31.154:25600`（**不要带结尾斜杠**） |
| 账号 | `test@local.test` |
| 库 | `JM` / `PICA` / `WNACG` |
| 语言 | 简体中文（`values-zh-rCN`，上游已有 928/928 完整翻译） |

部署侧的两个前提（`comic-komga` 里已实现）：

1. **`MEDIA.MEDIA_TYPE` 在入库时就写**（按扩展名），不再等分析。
   Komga 的 `MediaDto.mediaProfile` 完全由 `mediaType` 推导，为空时是老版本客户端崩的直接原因。
2. 按需分析开着（`komga.analysis.on-demand: true`），首次打开一本要读一次 115 上的压缩包
   （实测 9 页的书 6 秒），第二次走内存页缓存。

## 改动清单

### 1. 未分析的书不再崩「Unsupported book format」

`komelia-ui/.../reader/ImageReaderScreen.kt`

上游逻辑只认 `mediaProfile`：

```kotlin
mediaProfile == DIVINA || PDF || epubDivinaCompatible -> 图片阅读器
mediaProfile == EPUB -> EPUB 阅读器
else -> error("Unsupported book format")      // ← 未分析的书全走这里
```

我们的书大部分是「未分析」状态（`mediaStatus: UNKNOWN`、`pagesCount: 0`），于是**一本都打不开**。

改法：在 `else` 之前加一层按扩展名兜底（`cbz / cbr / cb7 / zip / rar / 7z`），
优先看 `book.url`（我们的书 `name` 里没有扩展名），再退回 `name`。
打开阅读器后 Komga 会自动按需分析，后续就拿到真正的 `DIVINA` 档案。

另外把错误信息补上书名与 mediaProfile，便于排查，不再是一句光秃秃的报错。

## 构建

需要 Android SDK + JDK 17/21：

```bash
./gradlew :komelia-app:assembleDebug
# 产物：komelia-app/build/outputs/apk/debug/*.apk
```

安装到平板（无线 adb）：

```bash
adb push komelia-app/build/outputs/apk/debug/*.apk /data/local/tmp/komelia.apk
adb shell pm install -r /data/local/tmp/komelia.apk
```

> 坑：小米设备的 `/sdcard` 是 fuse，`pm install /sdcard/xxx.apk` 会被 SELinux 拒
> （`system_server has no access to read file context u:or:fuse`），必须走 `/data/local/tmp/`。

## 本机构建实测（2026-11-22，redos）

```bash
# 环境
JDK 21 / Gradle 9.7.1 / AGP 9.1.1 / Kotlin 2.4.20
android-compileSdk = 37  →  sdkmanager 里包名是 "platforms;android-37.0"（不是 android-37）
```

首次构建 **28 分 48 秒**（252 tasks，KMP + Compose）。

### 两个必须知道的坑

**1. 子模块只认三个**：`:third_party:ChipTextField`、`:third_party:compose-sonner`、
`includeBuild("third_party/secret-service")`。递归拉全部子模块会卡在
`chromium.googlesource.com`（libwebp）——国内不通，Android 构建也用不到。只初始化这三个：

```bash
git submodule update --init --depth 1 \
  third_party/ChipTextField third_party/compose-sonner third_party/secret-service
# 若工作区是空的（子模块 git 有提交但没检出）：
git -C third_party/<name> checkout -f HEAD
```

**2. 光有 Kotlin 编译产物装上去会崩**：

```
UnsatisfiedLinkError: dlopen failed: library "libsqlitejdbc.so" not found
  at org.sqlite.core.NativeDB.<clinit>
  at snd.komelia.db.KomeliaDatabase.<init>
```

我们自己构建的 APK 只有 2 个 `.so`，官方包有 38 个——`libsqlitejdbc.so`、vips 那套编解码
（brotli / dav1d / de265 / exif / expat / ffi …）来自 native 子模块的 CMake 构建。
不编译 native 时，可直接复用**同版本官方 APK** 的 `.so`（源码同版本，ABI 一致）：

```bash
unzip -q Komelia-0.20.0.apk "lib/*" -d /tmp/native
for abi in arm64-v8a armeabi-v7a x86_64; do
  mkdir -p komelia-app/androidApp/src/main/jniLibs/$abi
  cp /tmp/native/lib/$abi/*.so komelia-app/androidApp/src/main/jniLibs/$abi/
done
./gradlew :komelia-app:androidApp:assembleDebug
```

APK 从 41 MB 变成 125 MB（四个 ABI 全含），装上去就能跑。

> debug 签名与官方发布签名不同，覆盖安装会 `INSTALL_FAILED_UPDATE_INCOMPATIBLE`，
> 需先 `pm uninstall`（会丢已登录的服务器配置）。

### MIUI 的 USB 安装弹窗

`pm install` 报 `INSTALL_FAILED_USER_RESTRICTED: Install canceled by user` 时，
是小米「USB 安装提示」弹窗在等确认，只给 7 秒。脚本里这样过：

```bash
( adb shell pm install -r /data/local/tmp/app.apk > /tmp/inst.log 2>&1 & )
sleep 3; adb shell input tap <继续安装 的坐标>
```

## 已知限制

- `BOOK.FILE_SIZE` 在本栈里曾全为 0（批量入库不 `stat`），书页会显示「0 B」——不影响阅读。
- 库很大（31 万系列），首次进入列表靠分页拉取，滚动会慢几秒。
