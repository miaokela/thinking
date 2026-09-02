# unidbg 与 Frida 实战案例 ×5（全流程详解）

> 5 篇完整的实战流程案例，覆盖**5 种不同的技术架构**。每篇都按统一模板展开：
> **抓包定位 → 静态搜字符串 → 动态 Frida → unidbg 终端报错 → 补齐环境 → Maven 打包 → 调用验证**。
> 目标均为泛化/代表型（"某短视频 App""某电商 App"…），技术细节可迁移到同类 App。仅供授权范围内学习与 CTF/自研分析。

---

## 案例一：某短视频 App —— Native 签名（X-Gorgon 类）

**架构**：核心签名在 `libttnet.so`（"六神"算法），Java 层只做传入，签名参数约 26 字节。

### 1. 抓包定位
请求头发现 `x-gorgon` 动态变化字段（26 字节），就是目标。

### 2. 静态找字符串
- **jadx 全局搜索**：`loadLibrary` → `System.loadLibrary("ttnet")`；`native` → `public native String getGorgon(...)`。
- **IDA 打开 `libttnet.so`**：`Shift+F12` 搜字符串 `Java_`，找到核心生成函数，记偏移（如 `sub_16CEA0`）与方法签名 `[B getGorgon([B)I`。

### 3. 动态 Frida
```js
Java.perform(function () {
    var M = Java.use('com.tt.app.MainActivity');
    for (var o of M.getGorgon.overloads) {
        o.implementation = function () {
            var a = Array.prototype.slice.call(arguments);
            console.log('args=' + JSON.stringify(a));
            var r = this.getGorgon.apply(this, arguments);
            console.log('ret=' + r);
            return r;
        };
    }
});
```
确认 `getGorgon(byte[]) → int`。

### 4. unidbg 终端报错
首次运行（`vm.setVerbose(true)`）终端打印：
```
JNI callStaticObjectMethod not implemented  class: android/provider/Settings$Secure  method: getString(Landroid/content/ContentResolver;Ljava/lang/String;)Ljava/lang/String;
```
> 含义：SO 在读取设备 `android_id`，unidbg 无真实设置库。

### 5. 补齐环境
在 `AbstractJni` 里补 `Settings$Secure.getString` 返回 16 位 hex（与 Frida 记录的设备一致）。再跑可能继续报 `getDeviceId`、`getSubscriberId`，逐一补。

### 6. 打包（Maven）
```xml
<dependency><groupId>com.github.zhkl0228</groupId><artifactId>unidbg-android</artifactId><version>0.9.7</version></dependency>
```
资源：`src/main/resources/` 放 `app.apk`、`libttnet.so`。

### 7. 调用
```java
DvmClass c = vm.resolveClass("com.tt.app.MainActivity");
DvmObject<?> obj = c.newObject(null);
DvmObject<?> ret = obj.callJniMethodObject(emulator, "getGorgon([B)I", byteArray);
```
输出与 Frida 一致则成功。

---

## 案例二：某电商 App —— 安全 SDK 签名（SecurityGuard 类）

**架构**：阿里系 `libsgmain.so` 生成 `x-sign`，安全强度高、回调 Java 频繁。

### 1. 抓包定位
请求头 `x-sign` 为动态字段。

### 2. 静态找字符串
- **jadx**：搜 `sgmain`、`SecurityGuard`、`sign`，找到调用安全 SDK 的类。
- **IDA**：`libsgmain.so` 里找 `RegisterNatives`（多为动态注册），看 `JNI_OnLoad` 注册表确定方法。

### 3. 动态 Frida
`Java.use('com.alibaba.wireless.security...')` hook 签名方法，确认入参（appKey、参数拼接串、时间戳）。

### 4. unidbg 终端报错（这类最典型）
```
JNI callObjectMethod not implemented  class: android/os/Build  method: getSerial()Ljava/lang/String;
...
JNI callStaticObjectMethod not implemented  class: com/example/H5Utils  method: getSign(...)
```
> 特点：既有系统类（Build）回调，又有**App 自定义 Java 类**回调——后者需要你从 jadx 反编译里复制并重写那段 Java 逻辑。

### 5. 补齐环境（逐个报错补）

原则：**让 SO 跑起来，报什么错就重写哪个回调**。下面两条报错分别对应**实例方法**和**静态方法**的回调。

#### 5.1 补 `android/os/Build#getSerial()`（实例方法回调）

报错：
```
JNI callObjectMethod not implemented  class: android/os/Build  method: getSerial()Ljava/lang/String;
```
`callObjectMethod` 是**实例方法**，重写 `callObjectMethodV`：

```java
@Override
public DvmObject<?> callObjectMethodV(VM vm, DvmObject<?> obj,
        String name, String signature, VaList vaList) {

    // 命中 android.os.Build.getSerial()，返回一个固定序列号
    if ("android/os/Build".equals(obj.getObjectClass().getName())
            && "getSerial".equals(name)) {
        return new StringObject(vm, "1234567890abcdef");
    }
    return super.callObjectMethodV(vm, obj, name, signature, vaList);
}
```

> 注意：`Build.getSerial()` 官方是 **static** 方法，正常情况下会走 `callStaticObjectMethodV`；但当 SO 通过 `jobject`（实例/对象指针）去调用它时，unidbg 会把它路由到 `callObjectMethodV`。**关键是根据实际报错走哪个回调就重写哪个**，别死记"它该是静态"。

如果你跑出来它实际走的是 `callStaticObjectMethodV`，就改为：

```java
@Override
public DvmObject<?> callStaticObjectMethodV(VM vm, DvmClass dvmClass,
        String name, String signature, VaList vaList) {
    if ("android/os/Build".equals(dvmClass.getName()) && "getSerial".equals(name)) {
        return new StringObject(vm, "1234567890abcdef");
    }
    return super.callStaticObjectMethodV(vm, dvmClass, name, signature, vaList);
}
```

#### 5.2 补 `com/example/H5Utils#getSign(...)`（App 自定义类，静态方法）

报错：
```
JNI callStaticObjectMethod not implemented  class: com/example/H5Utils  method: getSign(...)
```
这类是**业务自定义类**，unidbg 完全没有实现，必须**从 jadx 反编译里还原逻辑**。它返回的是 String，所以重写 `callStaticObjectMethodV`：

```java
@Override
public DvmObject<?> callStaticObjectMethodV(VM vm, DvmClass dvmClass,
        String name, String signature, VaList vaList) {

    if ("com/example/H5Utils".equals(dvmClass.getName()) && "getSign".equals(name)) {

        // 1) 从 vaList 读取入参（顺序、类型与方法签名一一对应）
        DvmObject<?> p0 = vaList.getObjectArg(0);      // 例：字符串参数
        DvmObject<?> p1 = vaList.getObjectArg(1);      // 例：对象参数
        String a = (String) p0.getValue();
        // ... 按需取其他参数

        // 2) 这里写从 JADX 里还原回来的真实 Java 逻辑
        //    例如：对参数做拼接、取 hash、再进 AES 等
        String sign = reproduceSign(a, dvmClass, vm);

        // 3) 返回 String，用 new StringObject 包装
        return new StringObject(vm, sign);
    }
    return super.callStaticObjectMethodV(vm, dvmClass, name, signature, vaList);
}

// 在 AbstractJni 子类里，用 JDK 类重现 H5Utils.getSign 的实现
private String reproduceSign(String input, DvmClass dvmClass, VM vm) {
    // 示例：按真实逻辑还原，这里只是示意
    // 实际是从 jadx 复制的 Java 代码（可用 StringBuilder / HashMap / Cipher 等 JDK 类）
    StringBuilder sb = new StringBuilder(input);
    sb.reverse();
    return sb.toString();   // 换成真实算法
}
```

**要点**：
- 返回 String → `new StringObject(vm, value)`；返回 int → `DvmInteger.valueOf(vm, value)`；返回 `byte[]` → `new ByteArray(vm, bytes)`。
- **入参读取**：`vaList.getObjectArg(i)`（对象）、`vaList.getIntArg(i)`（int）、`vaList.getLongArg(i)`（long）；参数顺序跟 `getSign(...)` 签名一致。
- **还原方式**：把 jadx 里 `H5Utils.getSign` 的 Java 源码"翻译"成上面 `reproduceSign` 的普通 Java 代码即可，不用理解 so。
- 若该方法还调用**其它 App 自定义类**，会继续报 `callXxx not implemented`，接着把那些类也这样补——**报一个补一个**。

#### 5.3 补不动时的退路

- 回调太深、业务类太多 → 改用 **JniForward**，把 unidbg 的 JNI 转发到真实 ART，省掉大量 override。
- 仍很麻烦 → **Frida + 真机**直接 hook，别硬啃 unidbg。

### 6. 打包
同前 Maven 结构，另需把 `libsgmain` 的依赖 so 也放进 resources。

### 7. 调用
按 `RegisterNatives` 注册的方法名+签名调用；若回调复杂到无法补齐，改用 **Frida + 真机**更合适。

---

## 案例三：某社交/IM App —— 加密函数 Hook 取 key + unidbg 复现

**架构**：登录/消息加密在 so 层，key 动态生成，需先 hook 拿到 key。

### 1. 抓包定位
请求体 JSON 里某个字段是密文（如 `enc_data`）。

### 2. 静态找字符串
- **jadx**：搜 `encrypt`/`AES`/`Cipher`/`loadLibrary`，定位加密工具类。
- **IDA**：找到 so 里的加解密函数（如 `sub_xxxx`）及 JNI 方法。

### 3. 动态 Frida（拿 key）
```js
Java.perform(function () {
    var C = Java.use('com.social.app.Crypto');
    for (var o of C.encrypt.overloads) {
        o.implementation = function () {
            var a = Array.prototype.slice.call(arguments);
            console.log('enc args=' + JSON.stringify(a));   // 可能含 key/iv
            return this.encrypt.apply(this, arguments);
        };
    }
});
```
抓到 key 与明文。

### 4. unidbg 终端报错
```
JNI callStaticLongMethodV not implemented  class: java/lang/System  method: currentTimeMillis()J
```
> SO 依赖系统时间戳，unidbg 缺 `System.currentTimeMillis`。

### 5. 补齐环境
```java
@Override
public long callStaticLongMethodV(VM vm, DvmClass dvmClass, String name, String signature, VaList vaList) {
    if ("java/lang/System".equals(dvmClass.getName()) && "currentTimeMillis".equals(name)) {
        return System.currentTimeMillis();   // 需与真机时间一致，否则签名对不上
    }
    return super.callStaticLongMethodV(vm, dvmClass, name, signature, vaList);
}
```

### 6. 打包
同前，含 `libcrypto.so`（加密库）。

### 7. 调用
用 unidbg 调 `encrypt`，传入 Frida 拿到的 key/明文，比对密文一致即成功。

---

## 案例四：某金融 App —— HTTPS + 双签名（SSL Pinning + 请求签名）

**架构**：既校验证书（SSL Pinning）又做请求体签名，抓包和复现都更难。

### 1. 抓包定位
抓不到 HTTPS → 先用 Frida 绕过 SSL Pinning。
```js
Java.perform(function () {
    var TrustAll = Java.use('javax.net.ssl.X509TrustManager')['$new'](Java.registerClass({
        name: 'T', implements: [Java.use('javax.net.ssl.X509TrustManager')],
        methods: { checkClientTrusted: function() {}, checkServerTrusted: function() {},
                   getAcceptedIssuers: function() { return []; } }
    }));
    var SSLContext = Java.use('javax.net.ssl.SSLContext');
    SSLContext.init.implementation = function (k, t, s) {
        return SSLContext.init.call(this, k, [TrustAll], s);
    };
    var P = Java.use('okhttp3.CertificatePinner');
    P.check.implementation = function () {};
});
```

### 2. 静态找字符串
- **jadx**：搜 `sign`/`md5`/`rsa`/`SM4`，定位签名工具；金融 App 常用**国密**。
- **IDA**：找 native 签名函数，可能是 SM4/RSA。

### 3. 动态 Frida
hook `sign` 确认入参（可能是 `timestamp + nonce + body` 拼接）。

### 4. unidbg 终端报错
```
JNI callStaticObjectMethod not implemented  class: java/security/MessageDigest  method: getInstance(...)
```
> SO 在用 `MessageDigest` 做摘要。

### 5. 补齐环境
`MessageDigest` 属 JDK 内置，可直接在 `AbstractJni` 里用 `java.security.MessageDigest` 实现：
```java
@Override
public DvmObject<?> callStaticObjectMethodV(VM vm, DvmClass c, String name, String sig, VaList va) {
    if ("java/security/MessageDigest".equals(c.getName()) && "getInstance".equals(name)) {
        return new DvmObject<>(vm, com.github.unidbg.linux.android.dvm.jni.ProxyDvmObject.CLASS, null);
        // 实践上常用 ProxyDvmObject 代理真实 JDK 对象，或重写 digest 相关回调
    }
    return super.callStaticObjectMethodV(vm, c, name, sig, va);
}
```

### 6. 打包
同前，含 `libsgmain`/`libcrypto` 等依赖。

### 7. 调用
先绕过 SSL（Frida），再把签名算法搬进 unidbg 复现；这类常需 **Frida + unidbg 联用**。

---

## 案例五：某小程序/轻量 App —— JS 加密 + 补环境（Node 侧）

**架构**：签名在 **JS 层**（小程序/H5），不涉及 so，用 Node 补环境复现。

### 1. 抓包定位
请求头/请求体加密字段（如 `sign`）。

### 2. 静态找字符串
- **Sources 面板全局搜索**：`encrypt`/`CryptoJS`/`MD5`/`sign`/`btoa`。
- 定位加密函数与混淆结构（数组混淆、控制流平坦化）。

### 3. 动态 Frida（浏览器/小程序环境 hook）
在 DevTools Console 重写：
```js
CryptoJS.AES.encrypt = function (msg, key, cfg) {
    console.log('key=' + key, 'msg=' + msg);
    return old(msg, key, cfg);
};
```
拿到 key/iv/明文。

### 4. Node 补环境报错（unidbg 的 JS 版）
把加密函数抽到 Node 跑，报：
```
ReferenceError: document is not defined
ReferenceError: navigator is not defined
```
> 缺浏览器环境对象。

### 5. 补齐环境
用 `jsdom` + 手工补 `window`/`navigator`/`canvas`/`localStorage`，或直接用 **Puppeteer/Playwright** 跑真浏览器（最省事）。
```js
const { JSDOM } = require('jsdom');
const dom = new JSDOM('<!DOCTYPE html>', { url: 'https://example.com' });
global.window = dom.window;
global.navigator = dom.window.navigator;
// 再 require 加密函数
global.document = dom.window.document;
```

### 6. 打包（Node 项目）
`package.json` 依赖：`jsdom` / `crypto-js` / `puppeteer`。

### 7. 调用
在 Node 里调用加密函数，用抓包/真机结果验证一致性。

---

## 5 篇案例对比速览

| 案例 | 架构 | 核心字段 | 典型报错 | 常用手段 |
|------|------|----------|----------|----------|
| 一、短视频 | Native 签名 | x-gorgon 类 | `Settings$Secure.getString` | unidbg 补 `Settings` |
| 二、电商 | 安全 SDK | x-sign 类 | `Build.getSerial` + App 自定义类 | unidbg + JniForward |
| 三、社交 IM | 加密 key | enc_data | `System.currentTimeMillis` | Frida 取 key + unidbg |
| 四、金融 | HTTPS+签名 | sign | `MessageDigest.getInstance` | Frida 过 SSL + unidbg |
| 五、小程序 | JS 加密 | sign | `document is not defined` | Node 补环境 / Puppeteer |

---

> ⚠️ 合规提示：以上 5 篇为**教学通用示例**，目标已泛化。请仅用于**授权范围内安全研究、CTF、自研/开源 App 分析**。对真实商业 App 的授权绕过、内容破解属违法行为，请勿用于此目的。
