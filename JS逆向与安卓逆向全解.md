# JS 逆向与安卓逆向全解

> 一份从零到进阶的系统化总结：把两大逆向方向涉及的知识点、难点、排坑技巧全部整理在一起。适合查漏补缺、复习、以及作为实战前的知识地图。

---

## 目录

1. [通用前置基础](#一通用前置基础)
2. [JS 逆向](#二js-逆向)
3. [安卓逆向](#三安卓逆向)
4. [通用逆向方法论](#四通用逆向方法论)
5. [工具链速查表](#五工具链速查表)
6. [学习与实战资源](#六学习与实战资源)

---

## 一、通用前置基础

两个方向共用的地基，缺了会寸步难行。

### 1.1 必会的技术栈

| 类目 | 知识点 | 说明 |
|------|--------|------|
| 网络协议 | HTTP/HTTPS、TCP/UDP、DNS | 看懂请求/响应、区别与应用层加密 |
| 抓包 | Charles、Fiddler、Burp、mitmproxy | 分析流量是逆向第一站 |
| 编程语言 | JavaScript、Python、Java、C/C++ | JS 逆向要懂 JS；安卓要懂 Java + C；自动化用 Python |
| 加解密 | 见下文「密码学基础」 | 辨认识别算法是核心能力 |
| 调试 | 断点、单步、Hook、内存查看 | 动态分析基本功 |
| Linux | 文件、进程、权限、shell | 安卓 native 层、脚本执行依赖 |

### 1.2 密码学基础（重中之重）

逆向里 80% 的活儿是"认出并还原加密算法"。

**常见算法速查**

| 算法 | 特征 | 典型场景 |
|------|------|----------|
| Base64 | 字符集 `A-Za-z0-9+/`，末尾可能 `=` 补齐，长度 4 的倍数 | 编码，非加密，易识别 |
| MD5 | 固定 32 位十六进制，无法解密（哈希） | 签名、摘要 |
| SHA1/SHA256 | 40/64 位十六进制 | 签名、指纹 |
| AES | 密钥 16/24/32 字节，ECB/CBC 模式，有 IV/PKCS7 | 数据加密（大量 App 用它） |
| DES/3DES | 64 位块，密钥 8/24 字节，弱（需换 3DES） | 老系统 |
| RSA | 非对称，公钥/私钥，常见 1024/2048 位 | 密钥交换、登录密码加密 |
| 国密 | SM2/SM3/SM4，国产 | 国内政务、支付类 App 常见 |
| XXTEA | 数据块加密，适合小数据 | 一些轻量 App |
| RC4 | 流加密，密钥长度可变 | 其他 |

**识别思路（顺序）**

1. 抓包看请求参数 → 猜测哪个字段是加密值
2. 看加密值长度/字符集 → 初步判断（32 位 hex 大概率 MD5；等长且带 `=` 可疑 Base64；密文定长 16 倍数可能是 AES/DES）
3. 在源码中搜索关键字：`AES`、`DES`、`CryptoJS`、`encrypt`、`MD5`、`Base64`、`padding`、`mode`、`iv`、`key`、`SM4`、`sm2.encrypt`
4. 定位到加密函数 → hook 拿到 `key/iv/明文/密文`

### 1.3 逆向前置思维

- **目标驱动**：先问"我要拿到什么？"，再决定是静态看、动态 hook 还是暴力还原。
- **终点思维**：加密最终会有一个"出口"（网络请求 / 内存 / 返回值），从出口往回倒推。
- **分层理解**：加密可能发生在 JS 层、Native 层（so）、或被加固壳包裹，逐层剥。

---

## 二、JS 逆向

JS 逆向主要发生在**浏览器端**（网页、小程序）和 **Node.js 环境**，目标是还原加密算法、签名、补环境。

### 2.1 核心技术栈

#### 2.1.1 抓包与定位加密

- **F12 Network**：看 XHR/Fetch 请求，找到可疑参数
- **XHR 断点 / 事件监听断点**：在 `send` 时断住，往回找参数组装点
- **Initator（调用栈）**：定位到 JS 源码的具体行
- **关键字搜索**：在 Sources 里全局搜 `encrypt`, `sign`, `token`, `md5`, `aes`, `CryptoJS`, `hex`, `base64` 等
- **Hook 重写**：在 Console 里重写 `XMLHttpRequest.prototype.send`、`fetch`、`document.cookie` 的 getter 来截获明文

#### 2.1.2 常用加密库识别

- **CryptoJS**：全局搜 `CryptoJS.enc.Utf8`、`CryptoJS.AES.encrypt`、`CryptoJS.MD5`
- **JSEncrypt**（RSA）：搜 `setPublicKey`、`JSEncrypt`
- **sm-crypto**（国密）：搜 `sm2`、`sm3`、`sm4`
- **forge**：搜 `forge`、`cipher`
- 常见库直接看 `node_modules` 对应名。

#### 2.1.3 混淆技术（难点）

| 类型 | 说明 | 对策 |
|------|------|------|
| 变量/函数名混淆 | `a.b.c()`, 十六进制命名 | 找逻辑线索、AST 还原 |
| 控制流平坦化 | 把代码改成 `switch + while` 状态机 | 还原为正常流程 |
| 字符串加密 | 字符串没了或变成 `String.fromCharCode` | 逐个解密 / hook `fromCharCode` |
| 数组混淆/自执行加密 | 代码被拆进数组 + 还原函数 | 执行还原函数拿到映射 |
| 反调试 | `debugger` 死循环、`console` 被禁用 | 关断点、hook `Function.prototype` |
| 特征关键字删除 | `encrypt` 等词被替换 | 靠行为/调用链定位 |

**反混淆技巧**

- 用 `AST` 解析 + `babel`/`escodegen` 重写，可配合 `webcrack`、`deobfuscator`
- 断点在混淆后的关键处单步看行为
- 对"解密函数"直接调用它，观察输入输出，不必理解全部代码

#### 2.1.4 补环境（Browser Environment）

很多加密依赖浏览器环境对象（`window`, `document`, `navigator`, `localStorage`, `Canvas`, `AudioContext` 等）。脱离浏览器跑 Node 时需要**补环境**。

- **要补的对象**：`window`、`document`、`navigator`、`location`、`localStorage`、`sessionStorage`、`screen`、`performance`、`canvas`、`WebGL`、`AudioContext`
- **常用补环境框架/工具**：
  - `jsdom`：模拟 DOM 环境
  - `puppeteer / playwright`：直接跑真浏览器（最省事，但慢）
  - `vm2` / `isolated-vm`：隔离 JS 执行
  - 社区成熟的补环境包：如 `js-cookie` 配套、`补环境框架`（如 `NodeJS 补环境` 教学库）
- **难点**：
  - 缺对象会报 `xxx is not defined`，需要**补齐缺失的属性和方法**
  - `navigator.webdriver`、`UserAgent`、`plugins`、`languages` 等指纹检测
  - `Canvas.toDataURL` 指纹、字体指纹

**补环境调试口诀**：报错什么补什么 → 先补最简单的对象 → 用 `__webdriver_evaluate` 之类检测是否有泄漏。

#### 2.1.5 指纹与反自动化对抗

- **User-Agent、Referer、Origin**：请求头校验
- **`navigator.webdriver`**：检测是否被 Selenium/Puppeteer 控制
- **`window.chrome`、`Object.getOwnPropertyDescriptor`**：检测浏览器可信度
- **Canvas / WebGL / Audio 指纹**：生成唯一标识
- **时间戳、随机数、请求序列**：防止重放
- **请求体签名**：常见 `sign(timestamp + nonce + secret)`

**绕过技巧**：在页面加载前注入 JS（`page.addInitScript`），重写 `navigator.webdriver`、补全 `window.chrome`、伪造 `window.outerWidth/Height`。

### 2.2 JS 逆向难点与技巧

#### 难点
1. 混淆严重时代码不可读
2. 加密 key 随机生成或在服务端下发，无法静态确定
3. 补环境对象多、版本差异大
4. 反调试/反自动化检测
5. 小程序、H5 与 Native 环境差异

#### 技巧
1. **Hook 优先于静态分析**：能直接 hook 拿 key/明文，就别死磕混淆代码。
2. **重写 `XMLHttpRequest` / `fetch`** 截获明文参数。
3. **用 `Object.defineProperty` 重写 `document.cookie`** 拦截 Cookie 设置。
4. **断点 + 调用栈**倒推加密入口，而非全文读代码。
5. **补环境用隔离 VM**，避免污染主环境。
6. **把加密函数单独抽出来**，在 Node 里 import 复现。

---

## 三、安卓逆向

安卓逆向目标：**Java 逻辑层** + **Native(So) 层** + **加壳/加固对抗**，最终实现协议还原或脱壳。

### 3.1 核心技术栈

#### 3.1.1 APK 结构与基础

- **APK = 压缩包**，包含 `AndroidManifest.xml`、`classes.dex`、`resources.arsc`、`lib/<abi>/xxx.so`、`assets/`
- **AAB**（Google Play 新格式）需转 APK 再分析
- **签名**：v1/v2/v3/v4，校验完整性，重打包必须重签
- **ClassLoader**：`PathClassLoader` / `DexClassLoader` / `InMemoryDexClassLoader`，加壳核心机制

#### 3.1.2 静态分析

| 工具 | 用途 |
|------|------|
| `jadx` / `JADX-GUI` | DEX → Java 反编译，首选 |
| `JEB` | 更强的反编译/脱壳 |
| `Apktool` | 反编译资源、smali、重打包 |
| `Android Studio` | 关联源码、查看字节码 |
| `GDA` / `CFR` | 备用反编译器 |
| `dex2jar` + `JD-GUI` | 老牌组合 |
| `strings` / `binwalk` | 关键词、找 so 线索 |

#### 3.1.3 动态分析

| 工具 | 用途 |
|------|------|
| `Frida` | 运行时 hook，目前主力，JS 脚本 |
| `Xposed` / `LSPosed` | 系统级 hook（需 root/框架） |
| `objection` | Frida 封装，快速注 hook |
| `Magisk` | root 环境、模块 |
| `ADB` | 设备交互、安装、日志 |

**Frida 常用脚本技巧**：
- `Java.perform(() => { ... })` 注入 Java 层
- `Java.use('类名')` 调用/修改类方法
- `Interceptor.attach` hook 原生函数
- hook `SSL` 相关函数绕过证书校验
- `Java.choose` 枚举实例 hook 对象方法

#### 3.1.4 Native / So 逆向

- **ARM 汇编**：ARM64（AArch64）必须，ARM32 也要懂
- **工具**：`IDA Pro`、`Ghidra`、`r2/radare2`
- **JNI 机制**：`JNI_OnLoad`、`Java_<包名>_<类>_<方法>` 命名规律、`RegisterNatives`
- **ELF 结构**：`.text`, `.rodata`, `.data`, `init_array`, 动态符号表
- **动态链接**：`dlopen`/`dlsym`、`DT_NEEDED`
- **so 混淆**：OLLVM（控制流平坦化 + 指令替换 + 虚假控制流 + 字符串加密）

#### 3.1.5 抓包与 SSL Pinning

- 工具：`Charles`、`Fiddler`、`Burp`、`mitmproxy` + 手机代理
- **问题**：App 校验证书（SSL Pinning）导致无法抓 HTTPS
- **绕过**：
  - 安装 CA 证书（用户+系统证书）
  - `Frida` hook `SSL_CTX_new` / `SSL_set_verify` / `X509_verify_cert` / `okhttp3` 的 `CertificatePinner`
  - `objection` 的 `android sslpinning disable`
  - `Magisk` 模块（如 `MagiskTrustUserCerts`）

#### 3.1.6 加壳 / 加固对抗

加固壳会在运行时动态解密/加载 DEX，导致静态分析失效。常见类型：

| 类型 | 说明 | 对抗 |
|------|------|------|
| DEX 整体加密 | 整个 classes.dex 加密，壳运行时解密 | Frida hook `DexClassLoader`、内存 dump |
| 抽取壳（函数抽取） | 关键 method 的 code 被清空，运行时回填 | hook `ClassLoader` / 内存 dump 补齐 |
| VMP | 字节码转自定义虚拟机指令 | 还原 VM / 动态分析 |
| So 加壳混淆 | native 层加密、OLLVM | unidbg 模拟执行、去混淆 |

**脱壳手段**：
- **内存 dump**：`frida` 脚本 `DumpDex`、`FART`、`BlackDex`、`dump-dex` 等
- **主动调用/类加载 hook**
- **unidbg**：在 PC 上直接模拟执行 So，无需真机加壳对抗

#### 3.1.7 unidbg（重点）

unidbg 是"基于 unicorn 的安卓模拟器"，能在 PC 上运行 So 文件，无需真实手机/加固环境。

- **核心库**：`com.github.zhkl0228:unidbg-android` / `unidbg-api`
- **用途**：
  - 直接调用 So 的 `JNI_OnLoad`、`Java_xxx` 方法
  - 绕过加壳/反调试，执行被抽离的 native 逻辑
  - 需要 Java 层环境时配合 `DvmClass`、`DvmObject` 模拟
- **基本流程**：
  1. 准备 so + 参考的 java 签名
  2. 继承 `AbstractJni` 模拟 `JNI` 回调
  3. 用 `DvmClass` 调用方法，拿返回值
- **难点**：JNI 模拟复杂、需要补齐 `java.*` 方法实现、签名/参数对不齐。

### 3.2 安卓逆向难点与技巧

#### 难点
1. 混淆 + 加固 + So 加密多层叠加
2. Frida/Xposed 被检测（反调试、反 Hook）
3. So 层算法还原（OLLVM 难读）
4. SSL Pinning 导致抓包失败
5. 协议签名算法复杂，需还原完整逻辑
6. 不同 ABI（arm64-v8a / armeabi-v7a）调用差异

#### 技巧
1. **先抓包 + 定位参数**，再决定从哪层下手。
2. **能用 Frida hook 就别硬啃混淆**。
3. **hook 顺序**：先找出加密/签名函数，再 hook 拿明文与 key。
4. **多 ABI 只留 arm64** 提高分析效率。
5. **脱壳后对比**：加固前/后 dex diff，找被抽函数。
6. **unidbg 做 So 算法复现**，脱离真机批量调用。
7. **重打包**：分析完可重签再测（注意 App 完整性校验）。

---

## 四、通用逆向方法论

### 4.1 逆向五步法

1. **抓包取证**：先看请求/响应，锁定目标参数（加密字段）。
2. **静态定位**：搜关键字（`encrypt`/`sign`/`md5`/`aes`…），找到疑似加密函数。
3. **动态验证**：Hook/断点，确认加密入口、拿到 key/iv/明文。
4. **算法还原**：复现加密逻辑（JS 抽函数 / Frida 复现 / unidbg 模拟）。
5. **协议模拟**：用 Python/Node 模拟完整请求，验证可复现。

### 4.2 加密算法识别决策树

```
加密值长度 + 字符集
 ├─ 32位hex → 大概率 MD5 → 确认是否加盐 → 还原拼接
 ├─ 40/64位hex → SHA1/SHA256
 ├─ 定长且 16 倍数 → AES/DES → 找 key/iv/mode/padding
 ├─ 非对称（很长, 512/1024位）→ RSA → 找公钥
 ├─ 含中文/特殊字节 → 可能 Base64 + 自定义
 └─ 每次变化 → 可能有随机数/时间戳参与
```

---

## 五、工具链速查表

### 5.1 JS 逆向工具

| 工具 | 用途 |
|------|------|
| 浏览器 DevTools | 断点、XHR、调用栈 |
| Charles / Fiddler / Burp / mitmproxy | 抓包 |
| CryptoJS | 加密复现 |
| jsdom | 补环境 |
| Puppeteer / Playwright | 真浏览器自动化、免补环境 |
| webcrack / deobfuscator | AST 反混淆 |
| e9t / babel | 代码改写 |

### 5.2 安卓逆向工具

| 类别 | 工具 |
|------|------|
| 反编译 | jadx、JEB、Apktool、GDA、dex2jar+JD-GUI |
| 动态 Hook | Frida、objection、Xposed/LSPosed、Magisk |
| So 分析 | IDA Pro、Ghidra、radare2 |
| 抓包 | Charles、Fiddler、Burp、mitmproxy |
| 脱壳 | FART、BlackDex、DumpDex、Frida 脚本 |
| 模拟执行 | unidbg、unicorn |
| 其他 | ADB、Android Studio、apksigner |

---

## 六、学习与实战资源

### 6.1 JS 逆向推荐路线

```
抓包 → 认识参数 → 搜关键字定位 → hook 拿明文
→ 复现加密 → 补环境(如需要) → 反混淆 → 模拟请求
```

### 6.2 安卓逆向推荐路线

```
Java基础 → 抓包 → jadx 静态分析 → Frida hook
→ So/IDA 逆向 → 加解密还原 → 协议还原
→ 加固/脱壳 → unidbg 模拟
```

### 6.3 练习素材

- **找真实 App/网站**：逆向自己常用的目标（注意授权）
- **CTF 逆向题**：BUUCTF、攻防世界、看雪 CTF
- **开源靶场**：如 `Android Crackme` 系列、`JS Crackme` 靶场
- **社区**：看雪论坛、吾爱破解、FreeBuf、先知社区

> ⚠️ 版权与合规：逆向分析请仅用于学习、安全研究或获得授权的目标。破解他人付费软件、传播盗版资源、绕过授权访问，均属违法违规行为。

---

## 附录：高频关键词速查

| 关键词 | 含义/线索 |
|--------|-----------|
| `encrypt` / `decrypt` | 加密/解密 |
| `sign` / `signature` | 签名 |
| `md5` / `MD5` | MD5 |
| `aes` / `AES` | AES |
| `des` / `DES` | DES |
| `base64` / `btoa` / `atob` | Base64 |
| `CryptoJS` | JS 加密库 |
| `JSEncrypt` | RSA |
| `sm2/sm3/sm4` | 国密 |
| `padding`, `mode`, `iv`, `key` | AES 参数 |
| `navigator.webdriver` | 自动化检测 |
| `RegisterNatives` | JNI 动态注册 |
| `JNI_OnLoad` | JNI 入口 |
| `ClassLoader` | 加壳/类加载 |
| `SSL_CTX_new` | SSL Pinning 绕过 |
| `Java.use` / `Interceptor.attach` | Frida 常用 |
