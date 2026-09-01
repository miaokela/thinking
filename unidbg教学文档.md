# unidbg 教学文档：架构 · 补环境 · Hook · 排错

> 本教程面向**学习 ELF/MachO 文件格式、ARM 汇编、JNI 机制与 Android 原生库模拟**的开发者。
> 所有示例均使用 unidbg 自带的测试 `.so` 或你自己编译的 native 库，**不涉及任何商业 App 的逆向目标**。

---

## 目录

1. [项目是什么](#1-项目是什么)
2. [架构与模块](#2-架构与模块)
3. [环境搭建](#3-环境搭建)
4. [快速入门：Hello unidbg](#4-快速入门hello-unidbg)
5. [核心概念速查](#5-核心概念速查)
6. [补环境详解](#6-补环境详解)
7. [Hook 详解](#7-hook-详解)
8. [调试与排错手册](#8-调试与排错手册)
9. [完整示例汇总](#9-完整示例汇总)
10. [实践练习](#10-实践练习)

---

## 1. 项目是什么

`zhkl0228/unidbg` 是一个**在 PC 上模拟执行 Android 原生库（`.so`）** 的工具，实验性支持 iOS Mach-O。它基于 Unicorn / dynarmic / hypervisor 提供 CPU 模拟，并完整实现了一套 **JNI（JavaVM / JNIEnv）**，让 native 库以为自己在真机上跑。

一句话概括：**它造了一个"假的 Android 环境"，让原生库不需要真机也能运行和调试。**

特点：

- 支持 ARM32 / ARM64
- 可调用 `JNI_OnLoad`，模拟 JNI 调用
- 支持 syscall 指令模拟
- 内置多种 hook：Dobby（inline hook）、xHook（Android 导入表 hook）、fishhook / whale（iOS）
- 提供调试器：单步、读写内存/寄存器、指令 trace
- 新版本支持 **MCP（Model Context Protocol）**，可接入 Cursor 等 AI 工具做辅助调试
- 提供内存泄漏检测、Worker Pool 等增强能力

> 官方定位：**"This is an educational project to learn more about the ELF/MachO file format and ARM assembly. Use it at your own risk!"**

---

## 2. 架构与模块

仓库分四个主要模块：

| 模块 | 职责 |
|------|------|
| `unidbg-api` | 核心 API：`Emulator`、`Memory`、`Module`、`Symbol`、hook 抽象、调试器接口 |
| `unidbg-android` | Android `.so` 模拟：Dalvik VM、JNI、`AndroidResolver`、`XHookImpl`、`Dobby` |
| `unidbg-ios` | iOS 模拟：fishhook / substrate / whale、ObjC / Swift runtime |
| `backend` | CPU 后端实现：unicorn、dynarmic、Apple M1 hypervisor、Linux KVM |

**典型调用链**（Android）：

```
你的 Java 测试代码
   │
   ▼
AndroidEmulatorBuilder ──► AndroidEmulator（内存 + CPU 后端）
   │
   ▼
createDalvikVM() ──► VM（JavaVM / JNIEnv 的模拟）
   │
   ▼
loadLibrary(.so) ──► DalvikModule（ELF 加载 + 重定位）
   │
   ▼
callJNI_OnLoad(emulator) ──► 触发 JNI_OnLoad，注册 native 方法
   │
   ▼
resolveClass(...) / callStaticJniMethodObject(...) ──► 调用目标 native 方法
```

---

## 3. 环境搭建

### 3.1 依赖

- JDK 8+（建议 8 或 11；仓库 Jitpack workflow 用 8）
- Maven 3.6+
- 可选：ARM 交叉编译工具（用于自己编译示例 `.so`，见 9.3）

### 3.2 拉取与构建

```bash
git clone https://github.com/zhkl0228/unidbg.git
cd unidbg
mvn clean package -DskipTests
```

> 也可以直接通过 JitPack 引入依赖：

```xml
<dependency>
    <groupId>com.github.zhkl0228</groupId>
    <artifactId>unidbg-android</artifactId>
    <version>0.9.9</version>
</dependency>
```

### 3.3 测试库文件位置

unidbg 自带一批示例 `.so`，位于：

```
unidbg-android/src/test/resources/example_binaries/
    ├── arm64-v8a/
    └── armeabi-v7a/
```

本教程示例统一使用其中的 `libjnidispatch.so`（JNI 示例库）。

---

## 4. 快速入门：Hello unidbg

最完整的入门示例是仓库里的 [JniDispatch64.java](unidbg-android/src/test/java/com/sun/jna/JniDispatch64.java)。

下面是最小可运行骨架：

```java
package com.example;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.DvmObject;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.linux.android.dvm.jni.ProxyClassFactory;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class HelloUnidbg {

    private final AndroidEmulator emulator;
    private final DvmClass cNative;

    private HelloUnidbg() {
        // 1. 创建模拟器
        emulator = AndroidEmulatorBuilder.for64Bit()
                .setProcessName("com.example.hello")
                .build();

        // 2. 系统库解析器（SDK 级别决定加载哪些系统 .so）
        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));

        // 3. 创建 Dalvik VM
        VM vm = emulator.createDalvikVM();
        vm.setDvmClassFactory(new ProxyClassFactory());   // 自动补 JNI 环境
        vm.setVerbose(true);                              // 打印 JNI 调用日志

        // 4. 加载 .so 并调用 JNI_OnLoad
        DalvikModule dm = vm.loadLibrary(
                new File("unidbg-android/src/test/resources/example_binaries/arm64-v8a/libjnidispatch.so"),
                false);
        dm.callJNI_OnLoad(emulator);
        Module module = dm.getModule();

        // 5. 解析目标类
        cNative = vm.resolveClass("com/sun/jna/Native");
    }

    public void run() {
        DvmObject<?> version = cNative.callStaticJniMethodObject(
                emulator, "getNativeVersion()Ljava/lang/String;");
        System.out.println("version=" + version.getValue());
    }

    public void destroy() throws Exception {
        emulator.close();
    }

    public static void main(String[] args) throws Exception {
        HelloUnidbg app = new HelloUnidbg();
        app.run();
        app.destroy();
    }
}
```

**关键点**

| 代码 | 作用 |
|------|------|
| `AndroidEmulatorBuilder.for64Bit()` | 建 64 位 ARM 模拟器，也可 `for32Bit()` |
| `AndroidResolver(23)` | 指定 Android SDK 级别，决定系统库版本 |
| `createDalvikVM()` | 模拟 JavaVM/JNIEnv |
| `ProxyClassFactory` | **自动补齐** native 反射到的 Java 类 |
| `loadLibrary(..., false)` | 布尔值 = 是否自动注册所有导出符号 |
| `callJNI_OnLoad` | 触发初始化，注册 native 方法 |
| `callStaticJniMethodObject` | 调用某个静态 native 方法（返回对象） |

---

## 5. 核心概念速查

| 概念 | 说明 | 常用类 |
|------|------|--------|
| 模拟器 | 承载 CPU 后端、内存、进程环境 | `AndroidEmulator` |
| 内存 | 库加载、符号解析、分配 | `Memory` |
| 模块 | 一个已加载的 `.so` | `Module` / `DalvikModule` |
| 符号 | 函数/变量在模块内的地址 | `Symbol` |
| VM | JNI 世界（JavaVM/JNIEnv） | `VM` |
| DvmClass | 模拟的 Java 类 | `DvmClass` |
| DvmObject | 模拟的 Java 对象 | `DvmObject` / `StringObject` |
| 类解析器 | 按 SDK 级别解析系统库 | `AndroidResolver` |
| 代理工厂 | 自动生成缺失的 Java 类 | `ProxyClassFactory` |
| Hook | 拦截并改写函数执行 | `IxHook` / `Dobby` / `IWhale` |
| 调试器 | 单步、读写内存寄存器、trace | `emulator.attach(...)` |

**方法命名约定**

- `callStaticJniMethodObject` → 返回对象（`StringObject`、`ByteArray` 等）
- `callStaticJniMethodInt` → 返回 int
- `callStaticJniMethodLong` → 返回 long
- 方法签名用 **JNI 描述符**：`getNativeVersion()Ljava/lang/String;`

---

## 6. 补环境详解

### 6.1 什么是补环境

`.so` 运行时会通过 JNI **反调 Java 方法**：拿时间戳、读设备信息、发网络请求、读系统属性、触发加密算法等。真机上这些由 Android 系统提供；在 unidbg 里没有真实系统，凡是 `.so` 调用到的 **JNI 方法 / 系统符号 / Java 类** 都必须手动补齐——这就是"补环境"。

补环境分三个层面，按优先级：

1. **`ProxyClassFactory`（自动补 Java 类）**
2. **`registerSymbol`（补缺失的 native 符号）**
3. **`addJni` / `registerNative`（补齐 JNI 方法实现）**

### 6.2 方式一：ProxyClassFactory 自动补

```java
vm.setDvmClassFactory(new ProxyClassFactory());
```

native 代码通过 `FindClass` / `NewObject` 反射的类，如果找不到真实实现，`ProxyClassFactory` 会**动态生成代理类**，避免直接崩溃。这是默认要开的第一层。

### 6.3 方式二：registerSymbol 补缺失符号

当 native 库依赖某个外部函数但符号表里没有时，用 `registerSymbol` 把它的地址指向已有符号。典型例子 [QDReaderJni.java](unidbg-android/src/test/java/com/github/unidbg/android/QDReaderJni.java)：

```java
@Override
public void onLoaded(Emulator<?> emulator, Module module) {
    if ("libcrypto.so".equals(module.name)) {
        Symbol DES_set_key = module.findSymbolByName("DES_set_key", false);
        Symbol DES_set_key_unchecked = module.findSymbolByName("DES_set_key_unchecked", false);
        // 缺失的符号，指向已有符号地址
        if (DES_set_key_unchecked == null && DES_set_key != null) {
            module.registerSymbol("DES_set_key_unchecked", DES_set_key.getAddress());
        }
    }
}
```

注册方式：`memory.addModuleListener(this)`，在库加载完成后回调里补符号。

### 6.4 方式三：补齐 JNI 方法实现

当 native 代码调用某个 **Java 方法**（例如 `System.currentTimeMillis()`、业务回调）时，最常见的补法是自定义一个 `JniModule` / `DvmClass` 绑定。

以 `DvmClass` 上注册 native 方法为例：

```java
// 定义一个 DvmObject 子类来实现某个 Java 方法
DvmClass clazz = vm.resolveClass("com/example/SomeClass");

// 用 JniModule 注册"当 native 调用某 Java 方法时"如何处理
vm.addJni(emulator, new JniModule(emulator, vm) {
    @Override
    public DvmObject<?> callStaticObjectMethod(BaseVM vm, DvmClass dvmClass,
                                               String signature, VarArg varArg) {
        if ("android/os/SystemClock".equals(dvmClass.getName()) &&
            "elapsedRealtime()J".equals(signature)) {
            // 补一个固定的系统时间
            return DvmObject.valueOf(vm, System.currentTimeMillis());
        }
        return super.callStaticObjectMethod(vm, dvmClass, signature, varArg);
    }
});
```

> 注意：`JniModule` 的具体方法名随版本略有变化，核心思路是 **"拦截某一 JNI 调用，返回你构造的值"**。

**补环境最常见的三种 "缺"**

| 缺的东西 | 现象 | 解决办法 |
|---------|------|---------|
| Java 类不存在 | `ClassNotFoundException` | `ProxyClassFactory` 或手动 `resolveClass` |
| Java 方法没实现 | `NoSuchMethodError` / 返回空 | `JniModule` 里补该方法 |
| native 符号不存在 | `undefined symbol` | `registerSymbol` 补地址 |

---

## 7. Hook 详解

unidbg 支持三类 hook，对应不同粒度：

### 7.1 xHook：Android 导入表 hook（整体替换）

适合：替换某个库的导入函数，如 `malloc`、`free`、`getpid` 等。

```java
import com.github.unidbg.hook.ReplaceCallback;
import com.github.unidbg.hook.xhook.IxHook;
import com.github.unidbg.linux.android.XHookImpl;

IxHook xHook = XHookImpl.getInstance(emulator);

xHook.register("libjnidispatch.so", "malloc", new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, HookContext context, long originFunction) {
        int size = context.getIntArg(0);
        System.out.println("malloc(" + size + ")");
        context.push(size);                 // 压栈保存参数
        return HookStatus.RET(emulator, originFunction); // 继续执行原函数
    }

    @Override
    public void postCall(Emulator<?> emulator, HookContext context) {
        System.out.println("malloc ret=" + context.getPointerArg(0));
    }
});

xHook.refresh();   // 必须刷新才生效
```

**关键 API**

| 方法 | 作用 |
|------|------|
| `onCall` | 进入函数时调用，可改写入参，或直接 `HookStatus.LR(..., 0)` 短路返回 |
| `postCall` | 函数返回后调用，可读取返回值 |
| `HookStatus.RET(emulator, origin)` | 继续执行原函数 |
| `HookStatus.LR(emulator, value)` | 跳过原函数，直接返回指定值 |
| `HookContext.getIntArg(i)` / `getPointerArg(i)` | 读第 i 个参数 |

### 7.2 Dobby：inline hook（在指令层面改写）

适合：对某个函数做**精确跳转/拦截**，或对函数内部地址做 instrument。

```java
import com.github.unidbg.hook.hookzz.Dobby;
import com.github.unidbg.hook.hookzz.IHookZz;

IHookZz hookZz = Dobby.getInstance(emulator);

// 拦截并直接返回 0（跳过 free）
Symbol free = emulator.getMemory().findModule("libc.so").findSymbolByName("free");
hookZz.replace(free, new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, long originFunction) {
        System.out.println("inline hook free");
        return HookStatus.LR(emulator, 0);
    }
});
```

### 7.3 Dobby instrument（只观察不干预）

适合：打印参数，不影响执行流程，用于定位问题。

```java
Symbol newJavaString = module.findSymbolByName("newJavaString");
hookZz.instrument(newJavaString, new InstrumentCallback<RegisterContext>() {
    @Override
    public void dbiCall(Emulator<?> emulator, RegisterContext ctx, HookEntryInfo info) {
        Pointer value = ctx.getPointerArg(1);
        System.out.println("newJavaString value=" + value.getString(0));
    }
});
```

### 7.4 选择建议

| 场景 | 推荐 |
|------|------|
| 整体替换某个导入函数 | xHook |
| 精确控制某函数返回/跳转 | Dobby `replace` |
| 只观察参数、不影响执行 | Dobby `instrument` |
| iOS 目标 | whale / fishhook / substrate |

---

## 8. 调试与排错手册

### 8.1 常用调试开关

```java
vm.setVerbose(true);                      // 打印 JNI 调用分发
emulator.traceCode();                     // 指令追踪（开销大，慎用）
emulator.traceRead(addr, size);           // 追踪内存读
emulator.traceWrite(addr, size);          // 追踪内存写
emulator.attach(DebuggerType.ANDROID_SERVER_V7); // 挂 gdb server
emulator.traceMemoryLeaks();              // 内存泄漏检测（try-with-resources）
```

### 8.2 查看内存 / 字节

```java
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.unidbg.utils.Inspector;

Symbol sym = module.findSymbolByName("malloc", true);
MemoryBlock block = memory.malloc(0x10, false);
Number ret = sym.call(emulator, 0x10);        // 调用符号
Pointer ptr = UnidbgPointer.pointer(emulator, ret.intValue() & 0xffffffffL);
Inspector.inspect(ptr.getByteArray(0, 0x10), "data");  // 打印 hex + ascii
block.free();
```

### 8.3 常见问题速查表

| 现象 | 原因 | 解决 |
|------|------|------|
| `ClassNotFoundException` | 类名/包名解析失败 | 用 `/` 分隔：`com/sun/jna/Native`；开 `ProxyClassFactory` |
| `NoSuchMethodError` / 返回 null | 某 Java 方法没补 | 用 `JniModule`/`addJni` 补该方法 |
| `undefined symbol` | 依赖外部函数缺失 | `findSymbolByName` + `registerSymbol` 补地址 |
| `JNI_OnLoad` 崩溃 | 类结构或包名不对 | 核对 `resolveClass` 的包结构 |
| 某调用卡死/崩溃 | 某函数执行到未补的环境 | 用 xHook/Dobby 短路该函数 |
| 某指令不支持 | CPU 后端差异 | 换 `DynarmicFactory` / `HypervisorFactory` |
| 时间/随机数不一致 | 缺少系统时钟环境 | 在 `JniModule` 里补 `System.currentTimeMillis` 等 |
| 输出为空 | native 没被正确触发 | `setVerbose(true)` + `traceCode()` 看执行路径 |

### 8.4 后端选型

```java
import com.github.unidbg.arm.backend.DynarmicFactory;
import com.github.unidbg.arm.backend.HypervisorFactory;

AndroidEmulatorBuilder.for64Bit()
        .setProcessName("com.example")
        .addBackendFactory(new DynarmicFactory(true))  // 允许缺失符号抛错
        // .addBackendFactory(new HypervisorFactory(true)) // Apple M1 最快
        .build();
```

> `true` = 找不到符号时抛异常（有助于尽早暴露缺失环境）；`false` = 静默跳过。

---

## 9. 完整示例汇总

以下是可直接研读的仓库完整示例：

| 示例 | 文件 | 知识点 |
|------|------|--------|
| JNI 调用 + 多种 hook + attach 调试 | `JniDispatch64.java` | 入门全套 |
| 补符号（registerSymbol） | `QDReaderJni.java` | 缺失符号补地址 |
| 参数传入 Map + ProxyDvmObject | `SignUtil.java` | 传复杂对象 |
| WhatsApp 类库 + MCP 工具 | `Utilities64.java` | MCP 自定义工具 |
| iOS IPA 加载 | `IpaLoaderTest.java` | iOS 模拟 |

### 9.1 JniDispatch64（精华摘录）

```java
// —— 建模拟器 ——
emulator = AndroidEmulatorBuilder.for64Bit()
        .setProcessName("com.sun.jna")
        .addBackendFactory(new HypervisorFactory(true))
        .build();

// —— 加载库 ——
VM vm = emulator.createDalvikVM();
vm.setDvmClassFactory(new ProxyClassFactory());
vm.setVerbose(true);
DalvikModule dm = vm.loadLibrary(
        new File("unidbg-android/src/test/resources/example_binaries/arm64-v8a/libjnidispatch.so"),
        false);
dm.callJNI_OnLoad(emulator);
module = dm.getModule();
cNative = vm.resolveClass("com/sun/jna/Native");

// —— xHook 替换 malloc ——
xHook.register("libjnidispatch.so", "malloc", new ReplaceCallback() {
    @Override public HookStatus onCall(Emulator<?> emulator, HookContext context, long originFunction) {
        int size = context.getIntArg(0);
        context.push(size);
        return HookStatus.RET(emulator, originFunction);
    }
    @Override public void postCall(Emulator<?> emulator, HookContext context) {
        System.out.println("ret=" + context.getPointerArg(0));
    }
});
xHook.refresh();

// —— Dobby instrument ——
IHookZz hookZz = Dobby.getInstance(emulator);
Symbol newJavaString = module.findSymbolByName("newJavaString");
hookZz.instrument(newJavaString, new InstrumentCallback<RegisterContext>() {
    @Override public void dbiCall(Emulator<?> emulator, RegisterContext ctx, HookEntryInfo info) {
        System.out.println("value=" + ctx.getPointerArg(1).getString(0));
    }
});

// —— 调用 native ——
DvmObject<?> version = cNative.callStaticJniMethodObject(emulator, "getNativeVersion()Ljava/lang/String;");
System.out.println(version.getValue());
```

### 9.2 补环境 + 调用（QDReaderJni 精要）

```java
// 实现 ModuleListener 补缺失符号
public class QDReaderJni implements ModuleListener {
    @Override
    public void onLoaded(Emulator<?> emulator, Module module) {
        if ("libcrypto.so".equals(module.name)) {
            Symbol a = module.findSymbolByName("DES_set_key", false);
            Symbol b = module.findSymbolByName("DES_set_key_unchecked", false);
            if (b == null && a != null) module.registerSymbol("DES_set_key_unchecked", a.getAddress());
        }
    }

    // 用 xHook 短路 free
    IxHook xHook = XHookImpl.getInstance(emulator);
    xHook.register("libd-lib.so", "free", new ReplaceCallback() {
        @Override public HookStatus onCall(Emulator<?> emulator, long originFunction) {
            return HookStatus.LR(emulator, 0);   // 直接返回，不真正 free
        }
    });
    xHook.refresh();

    // 调用带 String 参数的 native 方法，返回 byte[]
    ByteArray array = d.callStaticJniMethodObject(emulator,
            "c(Ljava/lang/String;)[B", new StringObject(vm, data));
    Inspector.inspect(array.getValue(), "c result");
}
```

### 9.3 自己编译一个测试 `.so`

推荐用 NDK 交叉编译一个极简库，避免依赖商业库：

```c
// jni/Example.c
#include <jni.h>
#include <stdlib.h>

JNIEXPORT jstring JNICALL
Java_com_example_Example_hello(JNIEnv *env, jobject thiz) {
    return (*env)->NewStringUTF(env, "hello from native");
}

JNIEXPORT jint JNICALL
Java_com_example_Example_add(JNIEnv *env, jobject thiz, jint a, jint b) {
    return a + b;
}
```

编译：

```bash
# 假设 NDK 在 $NDK
$NDK/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android21-clang \
  -shared -fPIC -I"$NDK/sysroot/usr/include" \
  -o libexample.so jni/Example.c
```

对应 Java 侧（unidbg 里）：

```java
vm.addJni(emulator, new JniModule(emulator, vm));  // 注册 JNI 环境
DalvikModule dm = vm.loadLibrary(new File("arm64-v8a/libexample.so"), true);
dm.callJNI_OnLoad(emulator);
DvmClass cls = vm.resolveClass("com/example/Example");

// 调用 add(1, 2) -> 3
int ret = cls.callStaticJniMethodInt(emulator, "add(II)I", 1, 2);
```

---

## 10. 实践练习

按难度递进，做完即掌握核心：

1. **Level 1**：用 `HelloUnidbg` 骨架加载 `libjnidispatch.so`，成功打印 `getNativeVersion`。
2. **Level 2**：实现对 `malloc` 的 xHook 替换，打印每次分配的 size 和返回值。
3. **Level 3**：实现 `ModuleListener`，用一个假符号 `registerSymbol` 补上缺失的依赖。
4. **Level 4**：写一个 `JniModule`，补一个 native 会调用的 Java 方法（如返回固定时间戳）。
5. **Level 5**：`emulator.traceCode()` + `attach(DebuggerType.ANDROID_SERVER_V7)`，单步观察一次调用。
6. **Level 6**：自编译 `libexample.so`，完成 `hello` / `add` 的模拟调用，并 hook 其中某个函数。

---

## 使用边界

unidbg 是公开的开源研究工具，用于学习 ELF / ARM / JNI 机制完全合理。请**不要**用它针对性地逆向商业 App（如短视频、社交平台）的签名算法来绕过其服务端校验——那属于攻破真实线上安全控制，违反平台条款。若用于教学与研究，请使用自带测试库或自编译的 native 库。
