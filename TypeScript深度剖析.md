# TypeScript 深度剖析：从原理到细节认知

> 本文不讨论"怎么写"，而讨论"为什么这样"。TypeScript 的特殊性在于它是**双层语言**：编译期是结构化的静态类型系统，运行期则是 JavaScript 引擎（V8）的动态世界。理解 TS 的钥匙，是同时看穿这两层，以及它们之间那道"类型擦除"的墙。
>
> 分析对象为 **TypeScript 5.x + V8 引擎**（Node.js / Deno / 浏览器共享的参考运行时，注明其他引擎差异处）。

---

## 目录

1. [类型擦除：理解 TS 的第一性原理](#1-类型擦除理解-ts-的第一性原理)
2. [结构化类型系统：鸭子的胜利](#2-结构化类型系统鸭子的胜利)
3. [`any` / `unknown` / `never`：类型格上的三个端点](#3-any--unknown--never类型格上的三个端点)
4. [类型收窄与控制流分析：编译器如何"读心"](#4-类型收窄与控制流分析编译器如何读心)
5. [泛型与型变：协变、逆变与"故意的不可靠"](#5-泛型与型变协变逆变与故意的不可靠)
6. [条件类型与 `infer`：类型层是图灵完备的](#6-条件类型与-infer类型层是图灵完备的)
7. [`interface` vs `type`：不只是风格之争](#7-interface-vs-type不只是风格之争)
8. [`enum` 的运行时真相：唯一的例外](#8-enum-的运行时真相唯一的例外)
9. [V8 对象模型：hidden class 与 inline cache](#9-v8-对象模型hidden-class-与-inline-cache)
10. [数字与数组的底层：Smi、elements kind](#10-数字与数组的底层smielements-kind)
11. [JIT 管线：Ignition → TurboFan 与去优化](#11-jit-管线ignition--turbofan-与去优化)
12. [`this` 绑定：调用点决定一切](#12-this-绑定调用点决定一切)
13. [原型链：`class` 到底是不是语法糖](#13-原型链class-到底是不是语法糖)
14. [作用域与闭包：`var/let/const`、提升与 TDZ](#14-作用域与闭包varletconst提升与-tdz)
15. [事件循环：宏任务、微任务与渲染时机](#15-事件循环宏任务微任务与渲染时机)
16. [Promise 原理与 `async/await` 状态机](#16-promise-原理与-asyncawait-状态机)
17. [`==` 与 `===`、`undefined` 与 `null`](#17--与--undefined-与-null)
18. [模块系统：ESM 与 CJS 的鸿沟](#18-模块系统esm-与-cjs-的鸿沟)
19. [`#private` 与 `private`：两个世界的私有](#19-private-与-private两个世界的私有)
20. [装饰器：两套标准的分裂](#20-装饰器两套标准的分裂)
21. [严格性开关：TS 的"完整形态"](#21-严格性开关ts-的完整形态)
22. [常见认知误区速查表](#22-常见认知误区速查表)

---

## 1. 类型擦除：理解 TS 的第一性原理

```typescript
function greet(name: string): string {
    return "hello " + name;
}
```

编译产物（JS）：

```javascript
function greet(name) {
    return "hello " + name;
}
```

**所有类型标注在编译后完全消失**——这叫类型擦除（type erasure）。由此推出 TS 世界的第一批铁律：

- **类型在运行时不存在。** `typeof x === "string"` 能工作，因为 `typeof` 是 JS 运算符，问的是运行时值；但你无法 `if (x is MyInterface)`——interface 在运行时没有任何对应物。
- **类型不影响运行时行为。** 改一个类型标注不会改变程序执行（唯一的例外是 `enum`、装饰器元数据、以及 `namespace` 这类会生成代码的"带电"语法）。这与 Java/C# 的泛型签名参与重载、反射截然不同。
- **运行时类型检查必须手写。** 校验外部数据（API 响应、配置文件）需要运行时验证器（zod、io-ts、手写类型谓词 `x is T`）——TS 类型标注对网络字节流没有任何约束力。
- **`as` 断言是"我发誓"，不是转换。** `x as string` 不会把数字变成字符串；它只是让编译器闭嘴，运行时原封不动。`as unknown as T` 的双重断言更是"我知道我在撒谎"。

**认知要点：** TS 是"带静态验证的 JavaScript"，不是"编译成 JS 的新语言"。所有 TS 类型问题的终极答案都要回到一句话：*擦除之后还剩什么？*

---

## 2. 结构化类型系统：鸭子的胜利

```typescript
interface Point { x: number; y: number }

class Vec2 {
    constructor(public x: number, public y: number) {}
}

const p: Point = new Vec2(1, 2);  // OK！没有任何 extends/implements
```

TS 用**结构类型（structural typing）**：两个类型兼容与否只看形状（成员及类型），不看名字和继承声明。这与 Java/C# 的名义类型（nominal typing）根本对立，与 Go 的 interface 哲学一致。

推论与陷阱：

- **"多余属性检查"只在对象字面量直接赋值时触发：**

```typescript
const a: Point = { x: 1, y: 2, z: 3 };       // 报错：z 多余
const obj = { x: 1, y: 2, z: 3 };
const b: Point = obj;                        // OK！obj 的类型满足 Point 的形状
```
字面量走"新鲜度（freshness）"检查防拼写错误；经过变量一转手就退回纯结构比较。

- **少参数函数兼容多参数函数类型**（`(a: number) => void` 可赋给 `(a: number, b: string) => void`）——这不是迁就，是 JS 生态的真实语义：`arr.forEach(x => ...)` 的回调本来就会被传三个参数。
- **带 `private`/`protected` 成员的 class 退回名义比较**——这是 TS 留给你的"名义类型逃生舱"（品牌类型 branding 的黑魔法基础：给类型加一个独有的私有/符号字段，阻止形状相同的不同概念混用）。
- 结构类型意味着**类型之间没有"是"的关系，只有"兼容"的关系**。`Dog extends Animal` 在 TS 里只是"Dog 的形状至少包含 Animal 的形状"。

---

## 3. `any` / `unknown` / `never`：类型格上的三个端点

类型系统在数学上是一个**格（lattice）**：越往上越"什么都能装"，越往下越"什么都不剩"。

| 类型 | 位置 | 可赋给它 | 可用它做什么 |
|---|---|---|---|
| `any` | **格点之外**（作弊码） | 一切 | 一切（编译器完全弃权） |
| `unknown` | 真正的顶部 | 一切 | **什么都不能做**，直到收窄 |
| `never` | 底部（空集） | 只有 never | 可赋给一切 |

关键区分：

- **`any` 是类型系统的逃逸舱，且有传染性。** `any` 参与的运算结果还是 `any`，一个 `JSON.parse` 的返回值可以让整条数据流的类型检查形同虚设。它"既兼容一切又被一切兼容"——这在理论上破坏了类型系统的一致性（unsound），是 TS 对 JS 现实的妥协。
- **`unknown` 是类型安全版的顶层类型。** 拿到 `unknown` 必须先用 `typeof`/`instanceof`/类型谓词收窄才能操作——`try/catch` 的 `catch (e)` 在严格模式下就是 `unknown`，逼你处理"JS 可以 throw 任何东西"的现实。
- **`never` 是"不可能的占位"，两大实战用途：**
  1. **穷尽性检查**：switch 的 default 里 `const _: never = value`，新增枚举成员时编译报错，强制你处理所有分支；
  2. **条件类型的"过滤删除"**：`T extends U ? X : never` 中 never 分支会让该成员从分布式结果里消失（见第 6 节）。

---

## 4. 类型收窄与控制流分析：编译器如何"读心"

TS 编译器做**控制流分析（control-flow analysis）**：沿着代码路径追踪每个变量的类型如何被判断语句收窄。

```typescript
function f(x: string | number) {
    if (typeof x === "string") {
        x.toUpperCase();  // 这里 x: string
    } else {
        x.toFixed(2);     // 这里 x: number —— else 分支也被"反向收窄"
    }
    x;  // 汇合点：恢复 string | number
}
```

收窄的手段清单：`typeof`、`instanceof`、`in`、`Array.isArray`、判别联合（discriminated union，靠共同的字面量字段 `kind: "a" | "b"`）、真值检查、自定义类型谓词（`x is T`）、`satisfies`/`as const` 辅助的字面量推断。

**三个高频反直觉点：**

1. **收窄不穿透函数边界。**

```typescript
const x: string | null = getValue();
if (x !== null) {
    setTimeout(() => x.length, 100);  // 报错！回调执行时 x 可能已被改回 null
}
```
编译器不假设闭包捕获的 `let` 变量在你收窄之后不会变——保守即正确。（`const` 变量无此问题，因为不可重新赋值，TS 5.4 起对 const 闭包收窄有进一步增强。）

2. **`null`/`undefined` 检查的别名会丢失收窄**（解构出 `const { name } = user` 后再判 `user !== null`，`name` 的类型不会联动收窄——5.5 起部分改善，但跨语句别名仍是收窄的边界）。

3. **判别联合是 TS 类型建模的核心武器**——用 `kind` 字面量字段替代 class 继承，是 TS 社区的"代数数据类型"实践，搭配 `never` 穷尽检查可获得接近 Rust `match` 的安全性。

---

## 5. 泛型与型变：协变、逆变与"故意的不可靠"

### 型变（variance）三问

若 `Dog <: Animal`（Dog 兼容 Animal），那么：

- `Array<Dog>` 兼容 `Array<Animal>` 吗？——**协变**（covariant）：是。
- `(x: Animal) => void` 兼容 `(x: Dog) => void` 吗？——**参数位置逆变**（contravariant）：方向反转才对。

TS 的答案：

- **数组/返回值：协变。** `Array<Dog>` 可赋给 `Array<Animal>`——这在理论上不安全（写入方可能塞入 Cat），但 TS 故意放行，因为"只读使用太常见，全禁会逼疯所有人"。这是**刻意的 unsound**。
- **函数参数：`strictFunctionTypes` 开启后是逆变**（安全方向），但**方法签名（method shorthand）仍然双变（bivariant）**——又一个为兼容 DOM 事件等既有模式留下的妥协。

```typescript
type Handler<T> = (x: T) => void;
const f: Handler<Animal> = (a: Animal) => {};
const g: Handler<Dog> = f;   // strict 下报错：逆变方向反了
// 但若写成 interface { onEvent(x: Animal): void } 方法形式，则可以
```

### 泛型是"擦除的、定义期检查的"

- TS 泛型**编译后完全消失**（无 Java 那样的 Class 对象，无 C# 那样的具现化 reification）——`T[]` 运行时就是普通数组，你无法 `new T()`，无法 `x instanceof T`。
- 泛型函数体在**定义处**对着约束（`T extends ...`）检查，与 C++ 模板在实例化时才检查相反——这是 TS 报错信息"稳定但保守"的原因：函数体内只能做约束允许的事。

```typescript
function first<T extends { length: number }>(xs: T): T {
    return xs;  // 函数体只能依赖 length: number 这个契约
}
```

---

## 6. 条件类型与 `infer`：类型层是图灵完备的

```typescript
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type A = UnwrapPromise<Promise<string>>;  // string
```

类型层的三件套：

- **条件类型** `T extends U ? X : Y`：类型层的 if。
- **`infer`**：在 extends 的模式里"捕获"子结构——本质是类型层的**模式匹配 + 变量绑定**。
- **递归类型**：条件类型可以引用自身（配合尾递归优化，TS 4.5+ 支持一定深度的尾递归条件类型）。

这三者组合，使 TS 的类型层被证明**图灵完备**——社区有人用类型实现了 JSON parser、SQL 解析器、甚至象棋引擎。实用推论：

- **分布式条件类型**：`T extends U ? ... : ...` 中若 `T` 是裸泛型参数且为联合类型，会**逐成员分发**再取并集——`Exclude<T, U>` 就靠这个实现。用 `[T] extends [U]`（包一层元组）可关闭分发。
- 官方工具类型全是这套机制的应用：`Partial`（映射类型 `in keyof`）、`Pick`、`ReturnType`（`infer` 函数返回值）、`Awaited`（递归解 Promise）。
- 模板字面量类型 `` `on${Capitalize<string>}` `` 让字符串操作进入类型层——事件名推导、路由参数提取（`"/user/:id"` → `{ id: string }`）皆源于此。

**代价认知：** 类型体操发生在编译器里，复杂度是有真实账单的——深层递归条件类型会让 tsc 变慢甚至 `Type instantiation is excessively deep` 报错。类型层的"能写"和"该写"是两回事，库作者炫技的债务由每个使用者的 IDE 响应速度偿还。

---

## 7. `interface` vs `type`：不只是风格之争

```typescript
interface User { name: string }
type User = { name: string }
```

表面等价，实质差异：

| 能力 | interface | type |
|---|---|---|
| 声明合并（declaration merging） | ✅ 同名自动合并（给第三方库类型"打补丁"的基础） | ❌ 重名即错 |
| 联合、元组、条件类型、映射类型 | ❌ | ✅ |
| `extends` 复杂计算类型 | 受限制 | 自由 |
| 报错信息可读性 | 显示名字 | 可能展开成巨大结构 |
| 性能细节 | 对象形状的缓存键更稳定，大型项目中 tsc 略快 | 联合/交叉大表达式更吃检查器 |

社区惯例：**对象形状优先 interface（可扩展、报错友好），其他一切（联合、函数、计算类型）用 type。**

---

## 8. `enum` 的运行时真相：唯一的例外

```typescript
enum Color { Red, Green }
```

编译产物：

```javascript
var Color;
(function (Color) {
    Color[Color["Red"] = 0] = "Red";
    Color[Color["Green"] = 1] = "Green";
})(Color || (Color = {}));
```

**enum 是 TS 中少数"带电"的语法**：它生成真实的运行时代码（一个双向映射对象：数字→名字、名字→数字）。这违反了类型擦除原则，带来一连串后果：

- `const enum` 不生成对象，成员在使用处**内联成字面量**——但在 `isolatedModules`（Babel/esbuild 单文件转译）下无法工作（转译器看不到别的文件里 enum 的值），现代工具链下属于陷阱。
- 数字 enum 有反向映射所以对象比想象的大；字符串 enum 没有反向映射。
- **现代替代方案：`as const` 对象 + 联合类型**，纯类型层、零运行时代码、可 tree-shake：

```typescript
const Color = { Red: "red", Green: "green" } as const;
type Color = typeof Color[keyof typeof Color];  // "red" | "green"
```

**认知要点：** 判断一段 TS 语法是否"安全"，先问它擦除后剩什么。enum、namespace、参数属性（`constructor(private x: number)`）、装饰器都会生成代码——它们是 TS 的"带电区"。

---

## 9. V8 对象模型：hidden class 与 inline cache

TS 编译成 JS 后，运行时的主角是 V8。理解它，才知道为什么有些 JS/TS 写法快、有些慢。

### Hidden Class（Map / Shape）

V8 不给对象用哈希表存属性（那太慢），而是为每个对象关联一个**隐藏类**：

```javascript
const p = {};
p.x = 1;   // 对象 shape: {} → {x}
p.y = 2;   // shape: {x} → {x, y}
```

每次属性增删，对象切换到 transition 树上的下一个 shape。**相同 shape 的对象，属性在固定偏移处，访问就是一次内存读取（接近 C struct）。**

推论：

- **构造时一次性建好所有字段**（TS class 的字段声明恰好鼓励这一点），不要在运行中增删属性；
- `delete obj.prop` 会把对象打回**字典模式（slow properties）**，性能塌方——置 `undefined` 代替；
- **同形状对象成组出现**（数组里放同一 class 的实例）是 V8 最擅长的场景。

### Inline Cache（内联缓存）

```javascript
function getX(o) { return o.x; }
```

第一次执行 `o.x`：查 shape，记下"shape A 的 x 在偏移 0"。之后若 shape 相同（**单态 monomorphic**），直接按缓存偏移读取，接近零成本；2–4 种 shape（**多态 polymorphic**）做几次比较；超过 4 种（**超态 megamorphic**）退回全局哈希表，慢一个数量级。

**性能铁律：让热路径上的对象保持同一种 shape。** 这就是为什么"参数类型统一"（哪怕 TS 允许联合类型）在性能敏感代码里是真实优化。

---

## 10. 数字与数组的底层：Smi、elements kind

### 数字：一个类型，两种表示

JS/TS 只有 `number`（IEEE 754 双精度），但 V8 内部区分：

- **Smi（small integer）**：31/32 位有符号整数，直接编码在指针里（tagged pointer），**无堆分配**；
- **HeapNumber**：超出 Smi 范围的整数和所有小数，堆上分配，参与运算要拆箱。

后果：整数运算快且零分配；一旦产生小数或超出 ±2³⁰，`number` 装箱上堆。`| 0`（强制转 32 位整数）这类古老 hack 就是给引擎的 Smi 暗示（现代代码请交给引擎，只在 profiling 后使用）。

### 数组：elements kind 迁移

V8 数组有 20+ 种内部形态，主线是：

```
PACKED_SMI ([1,2,3])  →  PACKED_DOUBLE ([1,2.5])  →  PACKED_ELEMENTS ([1,"a"])
        ↓ 出现空洞（arr[100] = 1）
HOLEY_*（带洞版本）→ 极端稀疏退化为 DICTIONARY（哈希表）
```

- **迁移只升不降**：`[1, 2.5]` 里删掉小数也不会回到 PACKED_SMI。
- 空洞数组（`new Array(10)`、`arr.length = 100`）让每次迭代都要检查原型链（空洞会向上查找！）——`for...of`/`forEach` 在 holey 数组上慢且有原型污染风险。
- **实践：数组初始化用字面量、保持类型单一、避免跳索引。** TS 的 `number[]` 标注对 V8 的内部形态**没有任何影响**——类型是编译期的，elements kind 是运行时的，两层互不通信。

---

## 11. JIT 管线：Ignition → TurboFan 与去优化

V8 执行 JS 的完整管线：

```
源码 → 解析 → 字节码（Ignition 解释器执行，同时收集类型反馈）
     → 热点函数（调用次数/循环回边超阈值）→ TurboFan 优化编译
     → 投机优化（speculative optimization）的本地机器码
     → 假设被打破 → 去优化（deopt），回退字节码
```

**投机是核心机制**：TurboFan 根据 inline cache 收集到的反馈打赌——"这个函数一直被 number 调用，那就生成 number 专用代码 + 类型守卫"。赌赢，快如 C；赌输（突然传入 string），**去优化**：丢弃优化代码，回到解释器重建状态。

推论：

- **多态和类型抖动是性能杀手**：同一个热函数今天收 `number` 明天收 `string`，TurboFan 反复 deopt，比不优化还慢。
- **TS 类型对 JIT 零帮助**：`x: number` 编译后消失，TurboFan 只能靠运行时反馈自己摸。这是 TS 与 Java 性能哲学的分野——**TS 的类型服务的是人，不是编译器**。
- 稳定 shape + 单态调用 + Smi 数组 = V8 的黄金组合；反过来，`eval`、`with`、`arguments` 滥用、`try/catch`（历史问题，现代 V8 已大幅改善）会关闭部分优化通道。

---

## 12. `this` 绑定：调用点决定一切

JS 的 `this` 不看**定义在哪**，只看**怎么调用**（箭头函数除外）。四条规则，优先级从低到高：

1. **默认绑定**：`f()` → 严格模式下 `undefined`，非严格模式全局对象；
2. **隐式绑定**：`obj.f()` → `this` 是 `obj`；
3. **显式绑定**：`f.call/apply/bind(that)` → 强制指定；
4. **new 绑定**：`new F()` → 新建对象。

**箭头函数不参与以上任何规则**：它没有自己的 `this`，词法捕获定义处的 `this`——这是它存在的核心理由。

```typescript
class Counter {
    count = 0;
    inc() { this.count++; }
}
const c = new Counter();
const fn = c.inc;
fn();            // this 丢失！隐式绑定在"摘下来"的瞬间失效
setTimeout(c.inc, 0);  // 同样的坑：回调被"裸调用"

// 解法三选一：
// 1. 箭头函数字段：inc = () => { this.count++ }   —— 词法 this，每个实例一份拷贝
// 2. 构造器里 this.inc = this.inc.bind(this)      —— 显式绑定
// 3. 调用处包箭头：setTimeout(() => c.inc(), 0)
```

**TS 能帮你一半**：`noImplicitThis` 让编译器检查函数内 `this` 类型；函数签名可声明 `function f(this: Counter)` 伪参数约束调用方式。但"摘方法"的运行时坑，编译器管不了。

---

## 13. 原型链：`class` 到底是不是语法糖

```javascript
const a = {};
a.__proto__ === Object.prototype;  // 每个对象有 [[Prototype]] 链
```

属性查找沿 `[[Prototype]]` 链逐级向上，直到 null。函数有 `prototype` 属性（作为"将来实例的父链"），`new F()` 把实例的 `[[Prototype]]` 指向 `F.prototype`——**两个 "prototype" 概念别混淆**。

`class` 与手工原型写法的真实差异（不只是糖）：

- class **不提升**（声明前访问是 TDZ 报错，函数声明则提升）；
- class 体内自动严格模式；
- 方法**不可枚举**（手工 `F.prototype.m = ...` 是可枚举的，`for...in` 行为不同）；
- `super` 是真实的语法机制（依赖 `[[HomeObject]]` 内部槽），无法用原型写法完整模拟；
- class 必须以 `new` 调用。

继承链真相：`class B extends A` 建立**两条链**——实例链 `B.prototype → A.prototype`（方法继承）和构造器链 `B → A`（静态成员继承，`B.staticMethod` 能找到 `A.staticMethod` 的原因）。

**TS 层的补充：** `implements` 只是编译期契约检查，编译后无影无踪；`override` 关键字（4.3+）是纯编译期的"我确实想覆盖"声明确认，配合 `noImplicitOverride` 防父类改名导致的静默断链。

---

## 14. 作用域与闭包：`var/let/const`、提升与 TDZ

### 三种声明的本质区别

| | 作用域 | 提升行为 | 重复声明 |
|---|---|---|---|
| `var` | 函数作用域 | 提升并初始化为 `undefined` | 允许 |
| `let` | 块作用域 | 提升但留在 **TDZ**（访问即 ReferenceError） | 禁止 |
| `const` | 块作用域 | 同 let | 禁止 |

**TDZ（暂时性死区）的认知**：`let` 也提升（在作用域顶部建立绑定），只是绑定处于"未初始化"状态，首次执行到声明语句才激活。TDZ 存在的意义是让"声明前使用"从静默 bug（undefined）变成响亮的错误。

**`const` 锁的是绑定不是值**：`const obj = {}` 后 `obj.x = 1` 合法——与 Python 的"绑定 vs 修改"是同一个分界。想要值级不可变：`Object.freeze`（浅冻结）或 TS 的 `as const`（编译期深度只读 + 字面量化）。

### 闭包捕获的是变量，不是值

```javascript
const fns = [];
for (var i = 0; i < 3; i++) fns.push(() => i);
fns.map(f => f());   // [3, 3, 3] —— var 只有一个绑定，三个闭包共享

for (let j = 0; j < 3; j++) fns.push(() => j);  // [0, 1, 2] —— let 每轮迭代新绑定
```

`let` 在循环中**每次迭代创建新绑定**——这正是 Go 1.22 修复的同一个坑，JS 在 ES6（2015）就用同样的方案解决了。闭包 = 函数 + 其词法环境的引用；只要闭包活着，整条作用域链上的变量都无法被 GC——**闭包是 JS 内存泄漏的常见元凶**（挂在大对象上的事件回调尤其典型）。

---

## 15. 事件循环：宏任务、微任务与渲染时机

JS 是单线程（主线程）+ 事件循环模型。队列分两级：

```
执行一个宏任务（script、setTimeout 回调、IO 回调、事件回调）
  → 清空整个微任务队列（Promise.then、queueMicrotask、MutationObserver）
    → （浏览器）需要时渲染一帧
      → 取下一个宏任务……
```

关键认知：

- **微任务在当前宏任务结束后、下一个宏任务/渲染之前全部排空**——包括排空过程中新产生的微任务。所以 `Promise.resolve().then(...)` 无限自增殖会**饿死渲染和 IO**（等价于同步死循环），而 `setTimeout` 自增殖不会。
- `setTimeout(fn, 0)` 不是"立即"：是"下个宏任务"，且浏览器对嵌套超时有限流（>5 层后最小 4ms）。
- **Node 与浏览器的事件循环不同**：Node 有 phase（timers → poll → check……）和第三优先级 `process.nextTick`（比微任务还靠前，是"当前操作完成后立刻"）。`setImmediate` 在 check 阶段。跨平台代码不要依赖这些顺序。
- **长任务阻塞一切**：主线程跑 200ms 的同步代码，UI 冻结、事件堆积。拆分手段：`scheduler.yield()`、把重活丢给 Web Worker / `worker_threads`。

---

## 16. Promise 原理与 `async/await` 状态机

### Promise 是一台状态机 + 回调注册表

```
pending → fulfilled（不可逆）
        → rejected（不可逆）
```

- `then(onF, onR)` 注册回调并**返回一个新 Promise**——链式的本质：每个 then 造一个新状态机，其决议值来自回调的返回值（返回 Promise 则"收养"其状态，thenable 同化）。
- 回调**永远异步执行**（入微任务队列），即使 Promise 已决议——Promise 消灭了"回调同步/异步不确定"的 Zalgo 问题。
- 决议值穿透：then 没传对应回调时，值/错误沿链下传——这就是"链尾一个 catch 兜底"的原理。
- `unhandledrejection`：链尾无 catch 且 Promise 已拒绝，运行时报 warning（Node 现代版本默认**崩溃**）——悬浮 Promise 是生产事故来源，`void promise` 或显式 catch。

### async/await：Promise 的语法外衣 + 微任务断点

```typescript
async function f() {
    const a = await g();
    const b = await h();
    return a + b;
}
```

编译后是一台驱动 Promise 的**状态机**（与生成器同源）：每个 `await` 是一个挂起点，等价于"把后续代码注册为 then 回调后返回"。推论：

- **`await` 必然让出至少一个微任务刻度**——`await 1` 也是异步断点；
- `async` 函数**同步执行到第一个 await**，之后才异步——`async f(); console.log(1)` 里 f 的前半段在 log 之前跑；
- **错误即 rejection**：async 函数内 throw ⟺ 返回 rejected Promise，`try/catch` 能捕获 await 的错误是因为它等价于 then 的第二个回调；
- **串行陷阱**：`await g(); await h()` 是串行的；无依赖的并发要 `await Promise.all([g(), h()])`。这是 async 代码最常见的性能 bug；
- `for await...of` 消费异步迭代器（`Symbol.asyncIterator`），是流式数据（网络分块、数据库游标）的标准抽象。

---

## 17. `==` 与 `===`、`undefined` 与 `null`

### `==` 的隐式转换

`===` 比身份/值不转换；`==` 走一套复杂的抽象相等算法（ToPrimitive/ToNumber 级联）：

```javascript
0 == ""        // true（"" → 0）
null == undefined  // true（特例，仅此一对）
[] == ![]      // true（![] → false → 0；[] → "" → 0）
NaN == NaN     // false（NaN 不等于一切，包括自己）
```

社区铁律**永远用 `===`**，唯一被容忍的例外：`x == null` 同时命中 null 和 undefined（一个判断挡两种空）。判 NaN 用 `Number.isNaN`（全局 `isNaN` 会先转数字，`isNaN("abc")` 为 true 的坑）。更精确的同一性：`Object.is`（区分 `+0`/`-0`，NaN 等于 NaN）。

### undefined vs null

- `undefined`：**系统用语**——未赋值、缺失参数、不存在的属性、无返回值函数的返回值。它是一个全局属性（ES5 后不可写），同时也是一个类型。
- `null`：**程序员用语**——"这里故意为空"。它是 `typeof null === "object"` 的著名的、已无法修复的历史 bug（早期值标签设计残留）。

约定：读外部数据用 `??`（只对 null/undefined 生效）而非 `||`（对 0、""、false 全部误判）——`??` 的存在本身就说明"空值语义"曾被 `||` 误伤了多少次。

---

## 18. 模块系统：ESM 与 CJS 的鸿沟

| | ESM (`import`) | CJS (`require`) |
|---|---|---|
| 加载 | **静态**：依赖图编译期可分析 | 动态：运行时函数调用 |
| 绑定 | **活绑定（live binding）**：导入方看到的是导出方变量的实时值 | **值快照**：`module.exports` 的属性拷贝 |
| 时序 | 异步设计（顶层 await 合法） | 同步加载 |
| tree-shaking | 可静态摇树 | 基本不可能 |
| this/严格 | 模块自动严格模式 | 非严格 |

**活绑定**是 ESM 的精髓：`export let count = 0` 后导出方 `count++`，所有导入方立刻看到新值——循环依赖因此可以"半工作"（函数声明提升可用，let 绑定在 TDZ 中不可用）。

**互操作（interop）的坑：**

- ESM import CJS：Node 把 `module.exports` 作为 `default` 导出，named exports 靠 cjs-module-lexer **静态猜测**——动态构造 exports 的库会猜不中；
- CJS require ESM：历史上不可能（`ERR_REQUIRE_ESM`），Node 22+ 对无顶层 await 的 ESM 开放 `require(esm)`；
- **双包危机（dual package hazard）**：库同时发 ESM 和 CJS 两份，同一个类在两个世界各有一份，`instanceof` 跨边界失效、单例变双例。

TS 层对应物：`moduleResolution: "bundler" | "node16" | "nodenext"` 决定了扩展名要求、类型解析路径（`exports` 字段的 `types` 条件）——**写库不配置 `types` 条件导出，等于类型对一半用户不可见。**

---

## 19. `#private` 与 `private`：两个世界的私有

```typescript
class A {
    private x = 1;   // TS 关键字
    #y = 2;          // JS 语法（ES2022）
}
```

- **`private`（TS）**：纯编译期检查。编译产物里就是普通属性 `x`，运行时用 `obj["x"]` 或编译后的 JS 直接访问畅通无阻。它是"君子协定"。
- **`#y`（JS 标准）**：**运行时真私有**。存储在对象内部槽（语义上等价于 WeakMap 封装），类外任何手段都摸不到——包括子类、包括序列化（`JSON.stringify` 自动跳过）、包括 Proxy。

实践结论：库作者、需要真实封装边界的场景用 `#`；需要子类访问用 `protected`（TS）或包内约定。两者可混用但风格应统一。`#` 字段还带来一个免费福利：`#field in obj` 是标准的"品牌检查"写法，比 `instanceof` 更能抵抗跨 realm（iframe/worker）问题。

---

## 20. 装饰器：两套标准的分裂

TypeScript 存在**两套互不兼容的装饰器**：

| | 实验性装饰器（legacy） | 标准装饰器（TC39 Stage 3） |
|---|---|---|
| 开启 | `experimentalDecorators: true` | TS 5.0+ 默认 |
| 时机 | TS 自创（2015），先于标准 | ES 2022+ 标准 |
| 签名 | `(target, propertyKey, descriptor)` | `(value, context)`（context 携带 kind/name/access） |
| 元数据 | 配合 `emitDecoratorMetadata` 可反射**设计期类型**（Angular/NestJS 的 DI 根基） | 无此机制 |
| 能力 | 可改 descriptor、可参数属性 | 可返回替换值、可 addInitializer |

**认知要点：**

- NestJS/TypeORM/Angular 整套生态建立在 legacy 版上（依赖 `design:paramtypes` 元数据做依赖注入）——换标准版不是改个开关的事；
- 标准版装饰器是**类元素的就地替换器**，类型更安全，但没有"反射出参数类型"的通道（类型已擦除，元数据是编译器特权）；
- 装饰器求值顺序：表达式自上而下求值，**调用自下而上**（与 Python 一致）。

---

## 21. 严格性开关：TS 的"完整形态"

`"strict": true` 是一把伞，下面是你应当理解其原理的每一项：

| 开关 | 防的是什么 |
|---|---|
| `strictNullChecks` | 十亿美元错误：`null/undefined` 不再能赋给一切类型，必须显式处理——**TS 价值的一半在这一个开关里** |
| `strictFunctionTypes` | 函数参数改双变为逆变（方法除外） |
| `noImplicitAny` | 推断不出类型时不再静默给 any |
| `noImplicitThis` | 函数内 this 必须有类型 |
| `strictPropertyInitialization` | class 字段必须在构造时初始化（或 `!` 断言/联合 undefined） |
| `strictBindCallApply` | bind/call/apply 的参数类型检查 |

推荐叠加的两个非 strict 开关：

- **`noUncheckedIndexedAccess`**：`arr[i]` 的类型从 `T` 变为 `T | undefined`——把"数组越界返回 undefined"这一运行时真相纳入类型。开了它你会发现一半的下标访问都是隐患。
- **`exactOptionalPropertyTypes`**：区分"属性不存在"与"属性存在但值是 undefined"——`{ x?: number }` 是否接受 `{ x: undefined }` 的差别，对 `Object.assign`/展开合并场景是真实语义差异。

**历史债认知：** strict 家族默认关闭纯粹是为了 2012 年的迁移路径。新项目不开 strict，等于买了保险丝然后把闸拔了。

---

## 22. 常见认知误区速查表

| 误区 | 真相 |
|---|---|
| TS 类型能保护运行时 | 类型编译后全擦除；外部数据必须运行时校验 |
| `as` 是类型转换 | 是编译期撒谎，运行时零操作 |
| `any` 和 `unknown` 差不多 | unknown 使用前必须收窄；any 是类型系统的弃权票 |
| 接口同名会冲突报错 | interface 会自动声明合并（这可能是特性也可能是惊吓） |
| class 只是原型的语法糖 | 有 TDZ、不可枚举方法、super、强制 new 等实质差异 |
| 箭头函数是普通函数的简写 | 没有自己的 this/arguments，不能 new，是词法 this 载体 |
| `setTimeout(fn, 0)` 立刻执行 | 排进下一个宏任务；微任务（Promise）全排在它前面 |
| async 函数整体异步 | 同步执行到第一个 await 才让出 |
| `await` 串着写最直观 | 无依赖的 await 串行 = 人为串行化；用 Promise.all |
| `const` 声明的对象不可变 | 锁的是绑定；属性随便改 |
| TS 的 `private` 能防运行时访问 | 编译期君子协定；真私有是 `#` 字段 |
| enum 是零成本抽象 | 它生成运行时代码；零成本的是 as const + 联合类型 |
| TS 类型能加速 V8 | 类型被擦除，JIT 只信运行时反馈；稳定 shape 才有用 |
| `== null` 该禁用 | 它是 == 唯一被官方认可的用法（同时挡 null 和 undefined） |
| CJS 里 `import` 和 `require` 只是语法不同 | 活绑定 vs 值快照、静态 vs 动态、异步 vs 同步，语义不同 |
| 类型体操越炫越好 | 类型复杂度是编译性能的账单，由所有使用者的 IDE 支付 |

---

## 结语：一条主线

TypeScript 的一切困惑，都源于它刻意维持的**双层分离**：

1. **类型层是编译期的证明系统。** 它擦除得彻彻底底，运行时无影无踪；它结构化、刻意保留若干 unsound 的逃生门（any、数组协变、双变方法），因为它的目标不是数学完备，而是**描述已经存在的 JavaScript 世界**。评价一个 TS 设计的标准永远是"对 JS 生态的拟合度"，不是类型理论的优雅度。

2. **运行时层是 V8 的动态世界。** hidden class、inline cache、Smi、elements kind、投机优化与去优化——性能的真相全部在这里，而类型标注对此**零贡献**。想跑得快，去研究引擎的形状稳定性，而不是类型复杂度。

3. **两层的接口只有一句话：擦除之后还剩什么？** enum、装饰器元数据、参数属性会"带电"残留；interface、泛型、断言灰飞烟灭。养成对每个语法问这句话的习惯，TS 就从"一堆规则"变成"一个模型"。
