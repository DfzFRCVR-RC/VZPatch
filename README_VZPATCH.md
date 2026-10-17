# VZPatch: 严格禁止签名校验的免 Root 注入工具 (基于 LSPatch 架构优化)

## 背景与痛点
现有免 root 注入工具（如 LSPatch、NPatch 等）在进行集成模式（Integrated Mode）打包时，存在一个通病：
1. **强制绕过签名校验**：默认会将原 APK 作为嵌套压缩包（如 `assets/lspatch/origin.apk`）完整复制进输出的 APK 中。
2. **体积显著膨胀**：导致生成的 APK 体积几乎翻倍（包含原包 + 新包资源/代码）。
3. **徒增开销**：如果目标应用本身并没有做签名自校验，这种硬性的签名绕过与原包拷贝纯属徒增 APK 体积和运行时解压开销，毫无技术必要。

## VZPatch 解决方案
VZPatch 基于 LSPatch 优秀的代码架构进行了深度重构与功能扩展，新增了 **`--strict`（严格禁止签名校验）** 选项。

### 核心特性
1. **绝对不打包原包**：在 `--strict` 模式下，VZPatch **绝不**将原 APK 作为 `assets/lspatch/origin.apk` 嵌套进输出。输出 APK 体积与原 APK 基本持平（仅增加轻量级的注入代理 `metaloader`、`loader` 及 native 库 `liblspatch.so`，总计仅增几百 KB 至数 MB）。
2. **纯净的注入执行**：
   - 编译期：原 APK 的所有 `classes*.dex` 被直接提取并按顺序重命名（`classes2.dex`, `classes3.dex` 等）打包进输出 APK，`classes.dex` 留给 `metaloader` 入口。
   - 运行期：`LSPApplication` 检测到 `strictMode` 开启时，**跳过**原包解压缓存、**跳过** `classLoader` 切换、**跳过**所有 PackageManger/签名 Hook（`SigBypass`）。
   - 注入逻辑：直接利用当前 APK 的 `classLoader` 加载原代码并初始化 Xposed 模块环境。
3. **适用场景**：适用于**没有做签名自校验**的目标应用，彻底告别 APK 体积翻倍的烦恼。

---

## 命令行用法

### 1. 严格模式打包（推荐，适用于无签名自校验应用）
```bash
java -jar vzpatch.jar -s -m module.apk target.apk
```
* `-s` / `--strict`：开启严格模式。**完全不做任何签名校验处理，绝不复制原包进 APK，仅运行主注入程序**。
* `-m` / `--embed`：嵌入 Xposed 模块。
* `-k` / `--keystore`：自定义签名（可选，默认使用内置调试签名）。

### 2. 对比传统 LSPatch
- **传统 LSPatch**：`java -jar lspatch.jar -m module.apk target.apk` （输出 `target-xxx-lspatched.apk`，包含完整原包，体积翻倍）。
- **VZPatch 严格模式**：`java -jar vzpatch.jar -s -m module.apk target.apk` （输出 `target-xxx-vzpatched.apk`，无原包嵌套，体积不膨胀）。

---

## 架构修改点说明
1. **Share 层 (`PatchConfig.java`)**：新增 `strictMode` 布尔字段，序列化进 `config.json` 与 AndroidManifest 的 `lspatch` meta-data 中。
2. **Patch 模块 (`LSPatch.java`)**：
   - 新增 `-s / --strict` 参数。
   - 当 `strictMode` 开启时，跳过 `addNestedZip`，改为直接迭代原 APK 条目：`classes*.dex` 重命名追加，其他资源/so 按原压缩方式（STORED/DEFLATED）复制，自动剔除原签名文件（`.SF`, `.MF`, `.RSA` 等）。
3. **Loader 模块 (`LSPApplication.java`)**：
   - 当 `strictMode` 生效时，`createLoadedApkWithContext` 直接返回 `stubLoadedApk`，不解压 `origin.apk`，不修改 `sourceDir`。
   - `onLoad` 中直接跳过 `switchAllClassLoader()` 与 `SigBypass.doSigBypass()`，纯净加载 Xposed 模块。
