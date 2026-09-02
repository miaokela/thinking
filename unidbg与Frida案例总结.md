# unidbg 与 Frida 逆向案例总结（学习版）

> 本文档汇总了互联网上公开、广泛流传的逆向实战案例，逐案例附上**核心代码**与**遇到的问题/解决办法**，供学习参考。所有示例均面向**授权范围/教学研究**，涉及具体商业 App 的目标以"某 App/某平台"泛化表述。

---

## 目录

- [一、unidbg 案例总结](#一unidbg-案例总结)
- [二、Frida 案例总结](#二frida-案例总结)
- [三、通用实战方法论](#三通用实战方法论)
- [四、高频报错速查](#四高频报错速查)
- [五、参考资源](#五参考资源)

---

## 一、unidbg 案例总结

unidbg 的核心价值：**不依赖真机，在 PC 上模拟执行 Android 的 `.so`（Native 层）**，用于逆向签名/加密算法。所有案例都围绕"加载 so → 补 JNI 环境 → 调用目标函数 → 拿结果"这条主线。

### 1.0 unidbg 基础框架（通用模板，后面所有案例都基于它）

下面是一个可跑通的**最小完整类**，集合了所有案例共同需要的结构。

```java
package com.example;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.DvmObject;
import com.github.unidbg.linux.android.dvm.StringObject;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.io.IOException;

public class SignDemo extends AbstractJni {

    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    private final Memory memory;

    public SignDemo(String apkPath, String soPath, String processName) throws IOException {
        // 1. 创建 ARM64 模拟器；setProcessName 用于规避"按进程名校验"
        emulator = AndroidEmulatorBuilder.for64Bit()
                .setProcessName(processName)        // 进程名很重要，SO 可能通过 /proc/self/cmdline 校验
                .build();
        memory = emulator.getMemory();

        // 2. 设置系统类库解析器（resources 里只集成了 sdk19 / sdk23）
        memory.setLibraryResolver(new AndroidResolver(23));

        // 3. 创建 Android 虚拟机，传入 APK 可过掉签名/包名校验
        vm = emulator.createDalvikVM(new File(apkPath));
        vm.setVerbose(true);        // 开启 JNI 调用详情日志，便于补环境
        vm.setJni(this);            // 绑定 JNI 回调实现（关键）

        // 4. 加载目标 so；第二参 true = 自动执行 .init/.init_array/JNI_OnLoad
        DalvikModule dm = vm.loadLibrary(new File(soPath), true);
        module = dm.getModule();

        // 5. 大部分静态注册的 so 无需手动 callJNI_OnLoad；
        //    若 SO 是动态注册(RegisterNatives)则需要：
        // dm.callJNI_OnLoad(emulator);
    }

    // ---- 通过 DvmClass 调用实例方法（最常用）----
    public String funcJ() {
        DvmClass dvmClass = vm.resolveClass("com.example.MainActivity");
        DvmObject<?> obj = dvmClass.newObject(null);          // newObject(null) 表示无参构造
        DvmObject<?> ret = obj.callJniMethodObject(emulator, "j()Ljava/lang/String;");
        return ret.getValue().toString();
    }

    public int funcP(int arg) {
        DvmClass dvmClass = vm.resolveClass("com.example.MainActivity");
        DvmObject<?> obj = dvmClass.newObject(null);
        return obj.callJniMethodInt(emulator, "p(I)I", arg); // 方法签名直接在 so 里推导
    }

    public String funcInit(int arg) {
        DvmClass dvmClass = vm.resolveClass("com.example.MainActivity");
        DvmObject<?> obj = dvmClass.newObject(null);
        DvmObject<?> ret = obj.callJniMethodObject(emulator, "init(I)Ljava/lang/String;", arg);
        return ret.getValue().toString();
    }

    // ---- 通过函数偏移直接调用（跳过反编译）----
    public String callByOffset(String input) {
        StringObject so = new StringObject(vm, input);
        Number ret = module.callFunction(emulator, 0x4a28D6,   // IDA 里查到的函数偏移
                vm.getJNIEnv(),                                // JNIEnv 指针
                vm.addLocalObject(so));                        // 将 Java 对象注册为本地引用
        return (String) ret; // 按函数返回类型强转
    }

    public static void main(String[] args) throws IOException {
        String apk = "src/test/resources/a.apk";
        String so  = "src/test/resources/libj.so";
        SignDemo demo = new SignDemo(apk, so, "com.example");
        System.out.println(demo.funcJ());       // j() 返回固定值
        System.out.println(demo.funcP(1738911344));
        System.out.println(demo.funcInit(9999));
    }
}
```

> 这套模板几乎是所有 unidbg 案例的骨架，后面每个案例都是在此基础上"加补环境"或"改调用方式"。

**unidbg 的三个 trace（对静态逆向最友好）**：

| Trace | 作用 |
|-------|------|
| 指令级 trace | 跟踪每条指令的执行，还原算法 |
| 内存读写 trace | 追踪关键内存的写入/读取位置，找签名参数 |
| 函数调用 trace | 定位 JNI 调用点，判断补环境该补哪里 |

---

### 1.1 X-Gorgon 签名（抖音/TikTok 系）

**背景**：`X-Gorgon` 是保护 API 通信的核心签名参数，约 26 字节 = **固定前缀（6 字节，00 00 00 00 00 00）+ 动态哈希（20 字节）**，生成涉及多层加密与混淆。

**涉及 so**：`libttnet.so`、`libcms.so`、`libmetasec.so`（"六神"签名算法）。

**关键流程**：
1. IDA 定位核心生成函数（如 `sub_16CEA0`）
2. unidbg 加载 so 并 hook 该函数
3. 用 trace 跟踪 26 字节生成过程，拆解"前缀来源 + 哈希算法"

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| `X-Gorgon` 前缀永远是 `00000000` | 前缀应来自设备 IMEI / Android ID 哈希，但 `Settings.Global.getString()` 等系统 API 模拟不准 | 在 `AbstractJni` 中精确补 `Settings.Global`/`Constants` 返回值 |
| 直接报 `callStaticObjectMethod not implemented` | SO 调用了 `android/provider/Settings$Secure.getString()`，unidbg 不知道返回什么 | 重写 `callStaticObjectMethodV`，对 `Settings$Secure.getString` 返回固定的 16 位 hex 字符串 |

补 `Settings.Secure` 的示例代码：

```java
@Override
public DvmObject<?> callStaticObjectMethodV(VM vm, DvmClass dvmClass, String name,
        String signature, VaList vaList) {
    if (dvmClass.getName().equals("android/provider/Settings$Secure")
            && "getString".equals(name)) {
        return new StringObject(vm, "5e8a7b9c3d4e5f60");  // android_id，注意与真机一致
    }
    return super.callStaticObjectMethodV(vm, dvmClass, name, signature, vaList);
}
```

---

### 1.2 小红书 x-s / x-mini（Frida + unidbg 联用）

**背景**：`x-s` 及登录场景的 `x-mini` 是小红书签名参数，来自 Native 层。

**关键流程**：
1. **用 Frida 在真机确认** JNI 调用点、类名、方法签名与入参
2. 把探路结果搬到 unidbg 复现
3. 登录场景走"纯 unidbg 签名服务"，涉及**真线程调度**

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| 调用后结果不稳定/偶发错误 | 登录场景涉及多线程调度，unidbg 是**协作式伪多线程** | 参考"真线程调度"方案：用 `vm.getThreadDispatcher()` 手动触发切换，或 hook `pthread_create` 改为主线程同步执行 |
| 报缺 JNI 方法 | 需要补 Java 层环境 | 重写 `callStaticObjectMethodV` / `callObjectMethodV` 补齐对应类方法 |
| Frida 探到的签名和 unidbg 不一致 | 入参/上下文未对齐 | 确保 Frida hook 时记录的**入参顺序、类型、方法签名**与 unidbg 调用完全一致 |

---

### 1.3 闲鱼/淘票票 x-sign

**背景**：`x-sign` 是阿里系某平台签名参数，核心逻辑在 Native 层。

**关键流程**：
1. **内存读写追踪 + Debugger Hook** 捕获关键入参
2. 用 unidbg `trace` 追踪 `x-sign` 的读写位置
3. 复现后请求接口校验算法正确性

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| 用 unidbg 跑出来和真机不一致 | NB/SDK 对**设备信息**（IMEI/Android ID/MAC）敏感 | 逐项补齐 `getDeviceId`/`getSubscriberId`/`getMacAddress` 对应的 JNI 回调 |
| Dump dex 后仍少逻辑 | 关键函数在 so 层，Java 层只是入口 | 改用 unidbg 直接调 so，不依赖 Java 层 |

---

### 1.4 淘宝/某麦 SecurityGuard（libsgmain）

**背景**：阿里系 `SecurityGuard` 的 `libsgmain` 负责生成 `x-sign` 等，安全强度极高。

**关键流程**：
1. 构建完整 unidbg 执行环境
2. **大量补 Java 环境**（H5/签名相关 API）
3. 通过 unidbg 完整模拟复用

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| 工程量巨大，补环境很耗时 | `libsgmain` 内部混淆 + 自校验多，回调 Java 层频繁 | 用 `gemini + claude` 等辅助写 `AbstractJni` 回调；或改用 **JniForward** 直接转发到真实 ART 省掉大部分 override |
| 自校验检测到模拟环境 | 检查 内存布局 / 属性 / 文件 | 补 `IOResolver` 提供 `/proc/self/maps`、`/proc/self/status` 等 |

---

### 1.5 通用 So 签名（V2-Sign / 按偏移调用）

**关键点**：直接通过**函数偏移**调用，无需反编译还原全部逻辑。它服务于"你只想拿到签名、不想看算法"的场景。

```java
// module 已在 loadLibrary 后通过 getModule() 拿到
DalvikModule dm = vm.loadLibrary(new File("libnet_crypt.so"), true);
Module module = dm.getModule();
vm.setJni(this);

// 偏移 0x4a28D6 是 IDA 里查到的目标函数地址
Number number = module.callFunction(emulator, 0x4a28D6, args);
```

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| 崩溃/返回错误 | 函数偏移不准，或调用约定(NEED)不对 | 用 IDA 确认精确偏移 + 参数个数/类型，注意 `vm.getJNIEnv()` 是第一个参数 |
| `dlopen` 加载失败 | so 有依赖的其它库未加载 | 先 `loadLibrary` 其依赖 so，或把这些依赖 so 也放到 resources 里 |

---

### 1.6 补环境（Java 层）核心

最主流方式：**继承 `AbstractJni`，重写关键回调**，模拟任意 JNI 调用。哪些是"自动处理"、哪些要"自己补"，是补环境的判断核心。

| JNI 操作 | 是否自动 | 说明 |
|----------|----------|------|
| `FindClass`/`GetObjectClass` | ✅ 自动 | 从类映射表查找 |
| `NewStringUTF`/`GetStringUTFChars` | ✅ 自动 | 字符串 UTF-8/16 转换 |
| `NewByteArray`/`GetArrayLength` | ✅ 自动 | 数组管理 |
| `NewGlobalRef`/`DeleteLocalRef` | ✅ 自动 | 引用计数 |
| `ExceptionCheck`/`ExceptionClear` | ✅ 自动 | 异常处理 |
| `CallObjectMethod`/`CallStaticObjectMethod` | ❌ 需补 | 转发到你的 `AbstractJni` 子类 |
| `GetStaticObjectField`/`SetStaticObjectField` | ❌ 需补 | 字段读写 |

**需手动重写的常用回调**：

```java
@Override
public DvmObject<?> callObjectMethodV(VM vm, DvmObject<?> obj, String name, String signature, VaList vaList) {
    // 按方法名/签名返回固定值
    if ("getDisplayWidth".equals(name)) return DvmInteger.valueOf(vm, 1080);
    return super.callObjectMethodV(vm, obj, name, signature, vaList);
}

@Override
public DvmObject<?> callStaticObjectMethodV(VM vm, DvmClass dvmClass, String name, String signature, VaList vaList) {
    if ("android/os/Build".equals(dvmClass.getName()) && "getSerial".equals(name)) {
        return new StringObject(vm, "1234567890abcdef");
    }
    return super.callStaticObjectMethodV(vm, dvmClass, name, signature, vaList);
}

@Override
public int getStaticIntField(VM vm, DvmClass dvmClass, String name, String signature) {
    if ("android/os/Build$VERSION".equals(dvmClass.getName()) && "SDK_INT".equals(name)) {
        return 28;
    }
    return super.getStaticIntField(vm, dvmClass, name, signature);
}
```

**补环境口诀**：`报错什么就补什么`，先跑通再完善——**不要**在运行前预判所有缺失，让它跑起来告诉你缺什么。

**进阶：JniForward**——把 unidbg 的 JNI 直接转发到真实 Android ART，可省掉大量 `java/*` 的 override。适用于补环境繁琐、回调频繁的案例。

---

## 二、Frida 案例总结

Frida 的核心价值：**运行时注入与 Hook**，覆盖 Java 层 + Native 层，是动态逆向的主力。

### 2.1 SSL Pinning 绕过（最经典）

**背景**：App 校验证书（Certificate Pinning）导致 Charles/Burp 抓不到 HTTPS。Hook 点常在 `SSLContext.init`、`X509TrustManager`、`okhttp3.CertificatePinner`、`libssl.so` 的 `SSL_CTX_new` / `SSL_set_verify`。

**完整脚本（Java 层 TrustManager 替换法）**：

```js
Java.perform(function () {
    // 构造"全部信任"的 TrustManager
    var TrustAll = Java.use('javax.net.ssl.X509TrustManager')['$new'](Java.registerClass({
        name: 'com.example.TrustAll',
        implements: [Java.use('javax.net.ssl.X509TrustManager')],
        methods: {
            checkClientTrusted: function (chain, authType) {},
            checkServerTrusted: function (chain, authType) {},
            getAcceptedIssuers: function () { return []; }
        }
    }));

    // Hook SSLContext.init，把 TrustManager 换成 TrustAll
    var SSLContext = Java.use('javax.net.ssl.SSLContext');
    SSLContext.init.implementation = function (km, tm, sr) {
        return SSLContext.init.call(this, km, [TrustAll], sr);
    };

    // 再 hook okhttp3 的 CertificatePinner
    var CertificatePinner = Java.use('okhttp3.CertificatePinner');
    CertificatePinner.check.implementation = function (hostname, peerCertificates) {
        return;  // 直接不检查
    };
});
```

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| 装完证书仍抓不到 | 只搞了 `TrustManager`，没处理 `CertificatePinner` | 两个点都 Hook |
| `Java.use('okhttp3.CertificatePinner')` 报找不到 | App 的 okhttp 版本/混淆后类名变了 | 从 jadx 里找真实类名；或用 `Java.choose` + `Java.enumerateLoadedClasses` 匹配 |
| 加了脚本后 App 直接闪退 | 触发了检测/完整性校验 | 先绕过检测；或先 `frida -f` 冷启动 + `--pause` |

**通用 bypass 脚本**：`frida-ssl-pinning-bypass`（含 `universal-ssl-bypass-ultimate.js`，一次 Hook 多框架：trustmanager/okhttp/trustkit/nsurlsession 等）。

**Flutter 场景**：在 `libflutter.so` 中定位证书校验函数偏移，Hook 使其恒返回 `true`。

**对象化工具**：`objection` 的 `android sslpinning disable` 可一键处理常见场景。

---

### 2.2 Hook 加密函数 / 取明文与 key

**背景**：定位签名/加密函数后，直接 Hook 拿 key/明文/密文，免去静态还原。

**关键技巧**：用 `overloads` 枚举**全部重载**再逐一挂钩——加固壳常对 `int/byte[]/String` 做多版本重载，漏一个就漏 hook。

```js
Java.perform(function () {
    var Target = Java.use('com.example.Encryptor');
    var methods = Target.sign.overloads;      // 枚举所有重载
    for (var i = 0; i < methods.length; i++) {
        methods[i].implementation = function () {
            var args = Array.prototype.slice.call(arguments);
            console.log('[sign] overload#' + i + ' args=' + JSON.stringify(args));
            // 拿到入参后，可在函数开头读取入参(明文/key)
            var ret = this.sign.apply(this, arguments);
            console.log('[sign] ret=' + ret);
            return ret;
        };
    }
});
```

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| 只 hook 到部分调用 | 存在多版本重载，只 hook 了其中一个 | 必须用 `overloads` 枚举全部 |
| `sign` 方法名找不到 | 类/方法被混淆 | 在 jadx 中找真实名；或用 `Java.enumerateLoadedClasses` 打印类再匹配 |
| 拿到的 key 是加密的 | key 本身在 so 里二次加密 | 结合 unidbg 追到 so 层，或 hook so 内层函数 |

**原则**：能动态 dump 出来的，就别花时间静态分析；静态只留给只存在于 so 里的逻辑。

---

### 2.3 加固壳 / 脱壳对抗

**背景**：App 被加固（抽取壳/VMP/SO 壳），静态分析失效。

**关键案例要点**：
- **ELF dump 修复**：用 `ElfDumpFixer` 等工具，基于 Android linker 一键 dump 被加固的 so 并修复 SectionTable
- **mmap/mprotect 钓真实 SO**：加固壳先加载壳 so、运行时再解密真实 so。hook `mmap`/`mprotect` 监视内存，钓出隐藏的真实 SO 并 dump
- **识别异常 ELF**：遍历 `dlopen` 已加载模块时，若某 SO 的 `JNI_OnLoad` 地址**不属于该 SO 内存区间**，说明被壳指向真实 so —— 这是常见的"壳伪装"破绽

**启动方式**：

```bash
frida -U -f com.example.app -l test.js --pause
```

`--pause` 暂停启动，再让 IDA attach（`gdb` 模式），配合断点绕过字符串/行为检测。

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| Dump 出的 so 无法反编译 | 未修复 SectionTable / 指针已改 | 用 `ElfDumpFixer` 修复 |
| 找不到真实 so | 壳用 mmap/mprotect 运行时解密 | hook `mprotect`/`dlopen` 监控加载 |

---

### 2.4 反调试 / 反 Frida 检测绕过

**背景**：App 检测 Frida、Xposed、root、模拟器，检测到就崩溃。

**关键技巧——`pthread_create` 阻断检测线程**：

```js
var pthread_create = Module.findExportByName('libc.so', 'pthread_create');
Interceptor.attach(pthread_create, {
    onEnter: function (args) {
        var f = this.function; // 可记录被创建的线程
    },
    onLeave: function (retval) {
        // 若检测到是"检测线程"，可修改返回值为错误码，让线程根本启动不了
        // retval.replace(0);  视情况决定
    }
});
```

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| 注入后崩溃/退出 | Frida 被检测 | hook `pthread_create` 阻断检测线程，使其不触发崩溃函数 |
| 找不到正确 attach 时机 | 目标方法已执行/未执行 | 用 `-f` 冷启动 + `--pause` 控制时机 |
| 进程号找不到 | 确认目标进程 | 用 `frida-ps -Uai` 列出 Android 进程，再注入 |

---

### 2.5 恶意软件 / 样本分析与取证

**背景**：逆向恶意 Android 样本，还原攻击链路、盗取数据逻辑。

**完整链路**：

```
静态分析(jadx) → 定位关键类 → Frida 动态分析 → 抓取加密数据 → 绕过 Frida 检测 → 还原通信协议
```

**遇到的问题**：

| 现象 | 原因 | 解决办法 |
|------|------|----------|
| 恶意样本混淆很重 | 刻意对抗分析 | 优先 Frida 动态拿实时行为，减少静态分析 |
| 样本有反调试 | 检测调试器 | `frida -U -f 包名 -l test.js --pause` 惰性注入 |

---

## 三、通用实战方法论

### 3.1 工具分工

| 阶段 | 首选工具 | 说明 |
|------|----------|------|
| 抓包定位参数 | Charles/Fiddler/Burp/mitmproxy | 先确定加密字段 |
| Java 层静态 | jadx / JEB | 看 DEX 逻辑，找入口 |
| Java 层动态 | Frida | Hook 加密/签名函数 |
| Native 层静态 | IDA / Ghidra | 看 so 反汇编，定位函数 |
| 脱离真机执行 | **unidbg** | 复现 so 算法，批量调用 |
| 脱壳 | FART/BlackDex + Frida dump | 拿到真实 dex/so |

### 3.2 标准逆向五步

```
1. 抓包取证          → 锁定需要还原的加密/签名字段
2. 静态定位          → jadx/IDA 搜关键字(encrypt/sign/md5/aes)
3. 动态验证          → Frida hook 确认入口、拿到入参与 key
4. 算法复现          → unidbg 模拟 so / Node 补环境复现 JS
5. 协议验证          → 模拟完整请求，确认结果可复现
```

### 3.3 unidbg + Frida 的配合

- **Frida 做"探路"**：在真机确认哪条路径、哪个函数、哪个偏移被调用
- **unidbg 做"复现"**：把探路得到的信息搬到 PC 上，稳定批量跑
- 联合使用能大幅降低工作量：**Frida 负责"找"，unidbg 负责"跑"**

---

## 四、高频报错速查

### 4.1 unidbg 常见报错

| 报错 | 含义 | 处理 |
|------|------|------|
| `callStaticObjectMethod not implemented` | SO 调了 Java 静态方法，unidbg 不知返回什么 | 在 `AbstractJni` 里重写 `callStaticObjectMethodV` 返回对应值 |
| `callObjectMethod not implemented` | 同上，但为实例方法 | 重写 `callObjectMethodV` |
| `JNI environment ...` / 找不到方法 | 类名/方法签名对不上 | 核对 so 中的签名；用 `vm.verbose(true)` 看 JNI 日志 |
| `dlopen` 失败 / 缺 so | 依赖库未加载 | 先 `loadLibrary` 依赖，或补 resources 里的 so |
| 运行崩溃/结果错乱 | 偏移错、调用约定错、ABI 不匹配 | 用 IDA 核对偏移；只保留 arm64；确认调用约定 |
| `pthread_join` 卡死 | 伪多线程死锁 | hook `pthread_create`/`pthread_join`，或手动触发线程切换 |
| 结果比真机慢几十倍 | Unicorn 解释执行 | 换 `DynarmicFactory` 后端加速（部分特性不支持） |

### 4.2 Frida 常见报错

| 报错 | 含义 | 处理 |
|------|------|------|
| `Java.use('X')` 失败 | 类名混淆/未找到 | 用 `Java.enumerateLoadedClasses`/`Java.choose` 找真实类名 |
| `Error: overload` | 方法存在多个重载 | 改用 `method.overloads` 枚举 |
| 注入后崩溃/退出 | 被 Frida 检测 | hook `pthread_create` 阻断检测线程 |
| 抓不到 HTTPS | SSL Pinning | 同时 hook `TrustManager` + `CertificatePinner`，装 CA |
| `frida: unable to find process` | 目标进程名/包名不对 | `frida-ps -Uai` 列出进程确认 |

---

## 五、参考资源

- **核心工具仓库**：`zhkl0228/unidbg`、`frida/frida`、`SensePost/objection`
- **SSL Pinning 绕过脚本集**：`eros1sh/frida-ssl-pinning-bypass`（含 universal-ssl-bypass-ultimate.js）、`jinwooid/SSL-pinning-bypass`
- **反混淆插件**：`obpo-project/obpo-plugin`（OLLVM 恢复）
- **脱壳工具**：`IIIImmmyyy/ElfDumpFixer`、FART、BlackDex
- **官方教材**：《unidbg 逆向工程：原理与实践》（陈佳林），系统讲解加载 so、补环境、Hook/Patch、真实案例
- **社区**：看雪论坛（unidbg/Frida 系列实战帖）、吾爱破解、奇安信攻防社区、HackTricks
- **实战案例来源**：CSDN/知乎/B站 的"某 App 签名逆向"系列，公众号 `泡泡以安`、`Javajava算法`

---

> ⚠️ 合规提示：以上所有案例与技术请仅用于**授权范围内的安全研究、CTF、自研/开源 App 分析**。对真实商业 App 的授权绕过、内容破解属违法行为，请勿用于此目的。
