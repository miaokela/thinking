# Frida 逆向实战（原理 · 核心 · 使用）

> 一份系统的 Frida 学习文档：从**工作原理**到**核心概念**，再到**主要使用方式**与常见场景。适用于 Android / iOS / Windows / macOS / Linux 的运行时逆向。

---

## 目录

- [一、Frida 是什么](#一frida-是什么)
- [二、Frida 的工作原理](#二frida-的工作原理)
- [三、核心概念](#三核心概念)
- [四、安装与常用 CLI](#四安装与常用-cli)
- [五、主要使用方式](#五主要使用方式)
- [六、常见场景实战](#六常见场景实战)
- [七、使用技巧与排错](#七使用技巧与排错)
- [八、官方资源](#八官方资源)

---

## 一、Frida 是什么

**Frida** 是一个**动态代码插桩（Dynamic Instrumentation）工具包**，允许你在**正在运行的进程**中注入 JavaScript，实时地**查看、修改、拦截**函数调用、内存数据和行为。

核心特点：

- **跨平台**：支持 Android、iOS、Windows、macOS、Linux、桌面/移动应用
- **双语言**：用 **JavaScript** 编写脚本，在注入的进程中执行（底层是 V8/QuickJS 引擎 + 一个小的 native 桥）
- **免重编译**：不需要重新打包 App，运行时注入即可
- **架构无关**：支持 ARM/ARM64/X86/X64

一句话：**Frida 就是"运行时 hook 的神器"**——你写一段 JS，注入到目标进程，就能改写它的行为。

---

## 二、Frida 的工作原理

理解 Frida 需要知道它的分层结构：

```
┌─────────────────────────────────────────────┐
│  你的 JS 脚本（Hook 逻辑）                  │
├─────────────────────────────────────────────┤
│  Frida Gum JS 引擎 (V8/QuickJS) + JS API     │
├─────────────────────────────────────────────┤
│  Frida Gum 插桩框架 (native)                 │
│  - Interceptor(指令级拦截) / 内存读写       │
├─────────────────────────────────────────────┤
│  目标进程（被注入的 App / 进程）            │
└─────────────────────────────────────────────┘
```

### 2.1 注入与通信模型

Frida 有两种注入方式：

| 方式 | 说明 | 适用 |
|------|------|------|
| **attach** | 附加到**已运行**的进程 | 调试正在运行的程序 |
| **spawn**（`-f`） | 启动进程并在首条指令处暂停，注入后恢复 | 冷启动、需要早于目标逻辑 hook |

```
Frida CLI  ──►  Frida Server(设备端)  ──►  目标进程
     │                                        │
     └─────────  通过 pipe/socket 传输 JS  ◄───┘
```

- **设备端**需要一个 `frida-server`（Android/iOS 上）或 `frida-gadget`（嵌入式）
- PC 端的 `frida` CLI / Python 绑定与设备端通信
- JS 脚本在**目标进程内**运行，因此能直接访问和修改目标的内存/函数

### 2.2 插桩机制的底层

Frida 的 **Gum** 引擎通过：

- **`Interceptor`**：在目标函数入口/出口安装 trampoline（跳板），拦截并转交给你的 JS 回调
- **替换内存页**：用 `InterceptionListener` 拦截；底层实际是对函数开头的指令做 `patch`，跳到 Frida 的 stub，再跳回
- **`Stalker`**：基于动态二进制的**指令级跟踪**，可单步/记录每一条指令及其执行流

---

## 三、核心概念

### 3.1 模块与导出函数

- **`Module`**：表示一个已加载的库/可执行文件（如 `libc.so`、`libapp.so`）
- **`Module.findExportByName(name, symbol)`**：找到某个导出符号的地址
- **`Process.enumerateModules()`**：枚举进程已加载的所有模块

```js
// 找 libc.so 里的 getpid
var getpid = Module.findExportByName('libc.so', 'getpid');
```

### 3.2 Interceptor（指令级拦截）

拦截**native 函数**，在调用前（`onEnter`）和调用后（`onLeave`）做处理：

```js
Interceptor.attach(Module.findExportByName('libc.so', 'write'), {
    onEnter: function (args) {
        console.log('write called, fd=', args[0]);
        this.buf = args[1];     // 保存参数供 onLeave 用
    },
    onLeave: function (retval) {
        console.log('write returned', retval);
    }
});
```

### 3.3 内存操作

```js
// 读内存
Memory.readCString(ptr);          // 读 C 字符串
Memory.readByteArray(ptr, size);  // 读字节数组
Memory.readUtf8String(ptr);       // 读 UTF-8

// 写内存
Memory.writeByteArray(ptr, [0x01, 0x02]);
Memory.writeUtf8String(ptr, 'hello');
```

### 3.4 Java 层 Hook（Android 专用）

**`Java.use`**：拿到 Java 类的封装；**`Java.perform`**：进入 Java 虚拟机上下文；**`Java.choose`**：枚举已存在的实例。

```js
Java.perform(function () {
    var Cipher = Java.use('javax.crypto.Cipher');
    Cipher.getInstance.overload('java.lang.String').implementation = function (s) {
        console.log('getInstance(' + s + ')');
        return this.getInstance(s);
    };
});
```

### 3.5 回调（Callbacks）与 Native 函数调用

```js
// 直接调用 native 函数
var strlen = new NativeFunction(Module.findExportByName('libc.so', 'strlen'), 'int', ['pointer']);
strlen(Memory.allocUtf8String('hello'));
```

---

## 四、安装与常用 CLI

### 4.1 安装

```bash
# PC 端工具（frida CLI + Python 绑定）
pip install frida-tools
# 或用 pipx 隔离
pipx install frida-tools

# 快速验证
frida --version
```

**设备端**：
- **Android**：推 `frida-server`（同架构，例如 `arm64`）到设备并运行：
  ```bash
  adb push frida-server /data/local/tmp/
  adb shell chmod +x /data/local/tmp/frida-server
  adb shell /data/local/tmp/frida-server &
  ```
- **iOS**：通过 Cydia 安装 `frida`，或手动装 `frida-server`

### 4.2 常用 CLI 命令

| 命令 | 用途 |
|------|------|
| `frida -U -f <包名> -l script.js` | **spawn** 启动进程并注入脚本（冷启动） |
| `frida -U -p <pid> -l script.js` | **attach** 到指定进程 |
| `frida -U -l script.js <进程名>` | attach 到该进程名的进程 |
| `frida-ps -Uai` | 列出 Android 设备上所有进程 |
| `frida-trace -U -f <pkg> -i "open|read"` | 跟踪指定函数调用 |

**常用参数**：`-U` USB 设备、`-H <host>` 远程、`-R` 使用 REPL、`--pause` 启动即暂停、`--runtime=v8/quickjs`。

> `-f` 冷启动让你有机会在**目标逻辑执行前**就 hook；`--pause` 后可用 IDA/GDB 再 attach。

---

## 五、主要使用方式

### 5.1 方式一：拦截 native 函数（`Interceptor`）

最常用、最通用。拦截任意 native 导出函数：

```js
Interceptor.attach(Module.findExportByName('libc.so', 'open'), {
    onEnter: function (args) {
        console.log('[open] path=' + Memory.readUtf8String(args[0]));
        this.fd = args[1];
    },
    onLeave: function (retval) {
        console.log('[open] fd=' + retval);
    }
});
```

### 5.2 方式二：Hook Java 方法（Android）

拦截 Java 层的加密/校验逻辑：

```js
Java.perform(function () {
    var M = Java.use('com.example.app.MainActivity');
    var encrypt = M.encrypt;
    for (var i = 0; i < encrypt.overloads.length; i++) {   // 枚举所有重载
        encrypt.overloads[i].implementation = function () {
            var a = Array.prototype.slice.call(arguments);
            console.log('[encrypt] args=' + JSON.stringify(a));
            var r = this.encrypt.apply(this, arguments);
            console.log('[encrypt] ret=' + r);
            return r;
        };
    }
});
```

### 5.3 方式三：`NativeFunction` 直接调用 native 函数

不依赖原函数被调用，自己主动调用：

```js
var system = new NativeFunction(Module.findExportByName('libc.so', 'system'), 'int', ['pointer']);
system(Memory.allocUtf8String('id'));
```

### 5.4 方式四：拦截系统调用（Syscall hook）

细粒度拦截，如 `openat`、`read`：

```js
var openat = new NativeFunction(Module.findExportByName('libc.so', 'openat'), 'int', ['int', 'pointer', 'int']);
Interceptor.attach(openat, {
    onEnter: function (args) {
        if (args[1].isNull()) return;
        console.log('[openat] ' + Memory.readUtf8String(args[1]));
    }
});
```

### 5.5 方式五：`frida-trace` 快速跟踪

无需写脚本，直接跟踪一组函数：

```bash
frida-trace -U -f com.example.app -i "open|read|write|recv|send"
```

### 5.6 方式六：Python 绑定（自动化）

Frida 可通过 Python 驱动，做批量/自动化 hook：

```python
import frida

def on_message(msg, data):
    print(msg)

# attach 目标进程
session = frida.get_usb_device().attach("com.example.app")
script = session.create_script(open("hook.js").read())
script.on("message", on_message)
script.load()
```

---

## 六、常见场景实战

### 6.1 SSL Pinning 绕过（Android 最经典）

```js
Java.perform(function () {
    // 构造"全部信任"的 TrustManager
    var TrustAll = Java.use('javax.net.ssl.X509TrustManager')['$new'](Java.registerClass({
        name: 'T',
        implements: [Java.use('javax.net.ssl.X509TrustManager')],
        methods: {
            checkClientTrusted: function () {},
            checkServerTrusted: function () {},
            getAcceptedIssuers: function () { return []; }
        }
    }));

    var SSLContext = Java.use('javax.net.ssl.SSLContext');
    SSLContext.init.implementation = function (km, tm, sr) {
        return SSLContext.init.call(this, km, [TrustAll], sr);
    };

    var P = Java.use('okhttp3.CertificatePinner');
    P.check.implementation = function () {};
});
```

### 6.2 取加密函数的 key/明文/密文（详见 5.2）

### 6.3 脱壳 / 找真实 so

- 通过 `Process.enumerateModules()` 遍历已加载 so，看 `JNI_OnLoad` 地址是否属于该 so 区间（壳伪装破绽）
- 用 `frida -U -f pkg -l dump.js --pause` 冷启动后再 dump

### 6.4 反调试 / 反 Frida 检测绕过

hook `pthread_create` 阻断检测线程：

```js
var pthread_create = Module.findExportByName('libc.so', 'pthread_create');
Interceptor.attach(pthread_create, {
    onEnter: function (args) {},
    onLeave: function (retval) { /* 可改返回值阻止线程启动 */ }
});
```

### 6.5 拦截并修改返回值

例如让校验函数永远返回 `true`：

```js
Java.perform(function () {
    var C = Java.use('com.example.Check');
    C.check.implementation = function () { return true; };
});
```

---

## 七、使用技巧与排错

| 问题 | 原因 | 对策 |
|------|------|------|
| `Java.use('X')` 失败 | 类名混淆/未加载 | 用 `Java.enumerateLoadedClasses()` 打印，或 `Java.choose` 枚举实例 |
| 报 `overload` 错 | 方法存在多个重载 | 改用 `method.overloads` 逐一遍历 |
| 注入后崩溃/退出 | 被 Frida 检测 | hook `pthread_create` 阻断检测线程 |
| 找不到进程 | 进程名/包名不对 | `frida-ps -Uai` 列出确认 |
| hook 不到早执行的方法 | 未用冷启动 | `-f` + `--pause`，在目标逻辑前注入 |
| `frida-server` 连不上 | 版本/架构不匹配 | 保证 PC 与设备的 `frida` 版本、架构一致 |
| GPU/多进程对不上 | App 多进程 | 给每个目标进程分别注入 |

**核心心法**：
1. **先 attach/spawn，再猜 hook 点**——用 `frida-trace` 或枚举定位
2. **能用运行时 hook 就拿到的，别硬啃静态分析**
3. **`-f` 冷启动 + `--pause`** 能在目标逻辑执行前介入，是最稳的做法
4. 对齐 **同一环境**（设备、时间、版本）否则结果对不上

---

## 八、官方资源

- **官网**：<https://frida.re>
- **源码**：`frida/frida`（GitHub）
- **文档**：<https://frida.re/docs/>
- **教程**：Frida 官方 `frida-examples`、`frida-code-snippets`
- **社区**：看雪论坛、HackTricks（Frida）、官方 Discord

---

> ⚠️ 合规提示：请仅对**授权范围内的 App、自研程序、CTF 靶场**应用 Frida。对真实商业 App 的授权绕过、内容破解属违法行为，请勿用于此目的。
