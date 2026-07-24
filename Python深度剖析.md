# Python 深度剖析：从原理到细节认知

> 本文不讨论"怎么写"，而讨论"为什么这样"。目标是把 CPython 的对象模型、内存管理、执行机制、以及那些容易踩坑的关键字（`global`、`nonlocal`、`yield`、`is`、`del`、装饰器、元类……）从原理上讲清楚。
>
> 除特别说明外，分析对象是 **CPython 3.11+**（参考实现），这是理解"Python 语言行为"的事实标准。

---

## 目录

1. [一切皆对象：对象模型的根基](#1-一切皆对象对象模型的根基)
2. [名字与绑定：Python 没有"变量"，只有"标签"](#2-名字与绑定python-没有变量只有标签)
3. [引用计数与循环垃圾回收](#3-引用计数与循环垃圾回收)
4. [内存管理的细节：小整数池、驻留、pymalloc](#4-内存管理的细节小整数池驻留pymalloc)
5. [GIL：为什么存在、意味着什么、正在发生什么变化](#5-gil为什么存在意味着什么正在发生什么变化)
6. [作用域与闭包：LEGB、cell、晚期绑定](#6-作用域与闭包legbcell晚期绑定)
7. [`global` 与 `nonlocal`：不是声明，是"绑定指令"](#7-global-与-nonlocal不是声明是绑定指令)
8. [`is` 与 `==`、`del` 的真实语义](#8-is-与--del-的真实语义)
9. [函数即对象：默认参数陷阱与 `*args/**kwargs`](#9-函数即对象默认参数陷阱与-argskwargs)
10. [装饰器：语法糖之下的真相](#10-装饰器语法糖之下的真相)
11. [迭代协议与生成器：`yield` 的本质是"可暂停的栈帧"](#11-迭代协议与生成器yield-的本质是可暂停的栈帧)
12. [`yield from` 与协程：生成器如何长成 async/await](#12-yield-from-与协程生成器如何长成-asyncawait)
13. [描述符协议：属性访问背后的总开关](#13-描述符协议属性访问背后的总开关)
14. [元类与 `type`：类也是对象](#14-元类与-type类也是对象)
15. [MRO 与 C3 线性化](#15-mro-与-c3-线性化)
16. [`__slots__`：省内存的代价](#16-__slots__省内存的代价)
17. [上下文管理器与 `with` 协议](#17-上下文管理器与-with-协议)
18. [异常机制：异常是对象，栈帧是代价](#18-异常机制异常是对象栈帧是代价)
19. [import 系统：模块是单例，循环导入为何出错](#19-import-系统模块是单例循环导入为何出错)
20. [字节码与执行模型：`dis` 看世界](#20-字节码与执行模型dis-看世界)
21. [拷贝语义：`copy` 与 `deepcopy`](#21-拷贝语义copy-与-deepcopy)
22. [常见认知误区速查表](#22-常见认知误区速查表)

---

## 1. 一切皆对象：对象模型的根基

在 CPython 中，**一切运行时实体都是一个 `PyObject`**：整数、字符串、函数、类、模块、甚至 `None` 和类型本身。每个对象的 C 层结构至少包含两个字段：

```c
// 简化版
typedef struct _object {
    Py_ssize_t ob_refcnt;   // 引用计数
    PyTypeObject *ob_type;  // 指向类型对象（决定这个对象"是什么"）
} PyObject;
```

由此推出几个关键认知：

- **对象有三要素：身份（id，即内存地址）、类型（type）、值（value）。** `id()` 在 CPython 中就是对象的内存地址。
- **类型本身也是对象。** `type(int)` 是 `type`，`type(type)` 还是 `type`——`type` 是自己的类型，这是元类体系的闭环点。
- **可变与不可变的区别发生在 C 层。** 不可变对象（int、str、tuple、frozenset）创建后值不可改，任何"修改"都会产生新对象；可变对象（list、dict、set）允许原地修改。
- **小对象可能被你"看见"复用。** `a = 1000; b = 1000; a is b` 的结果在交互式和脚本中可能不同——这不是语义，是实现细节（编译单元内的常量折叠）。**绝不要依赖它。**

一个经典实验：

```python
>>> a = 256; b = 256
>>> a is b
True          # 小整数缓存 [-5, 256]
>>> a = 257; b = 257
>>> a is b
False         # 脚本中；交互式逐行编译时可能为 True
```

**认知要点：** `is` 比较的是身份，`==` 比较的是值（调用 `__eq__`）。语义正确的比较几乎总是 `==`，`is` 只应用于单例（`None`、枚举、哨兵对象）。

---

## 2. 名字与绑定：Python 没有"变量"，只有"标签"

Python 的赋值 `a = [1, 2, 3]` 的正确心智模型是：

1. 在堆上创建一个 list 对象；
2. 把名字 `a` 绑定到该对象（在当前命名空间的字典里插入一条 `{"a": <对象指针>}`）。

由此直接解释三个"诡异"现象：

```python
a = [1, 2]
b = a          # b 绑定到同一个对象，不是拷贝
b.append(3)
print(a)       # [1, 2, 3]

x = 5
def f():
    x = 10     # 函数内出现赋值 => x 在整个函数作用域内是局部名字
    print(x)
f()
print(x)       # 5，互不影响

def g():
    print(x)   # UnboundLocalError！因为后面有赋值，x 被判定为局部名
    x = 1
```

第三条是经典面试题：Python 在**编译期**扫描函数体，只要函数内任何地方对某名字有赋值，该名字在整个函数内就是局部的——于是赋值前的读取报 `UnboundLocalError`，而不是读全局值。

**认知要点：** 赋值是"贴标签"，传参是"贴新标签"（共享引用，即 call by object reference）。函数内对不可变对象"重新赋值"无副作用，对可变对象"原地修改"影响调用方——这不是两套传参机制，是同一套机制下可变/不可变的自然结果。

---

## 3. 引用计数与循环垃圾回收

CPython 的内存管理是 **引用计数为主 + 分代循环检测为辅** 的双层机制。

### 引用计数

- 每个对象头里存 `ob_refcnt`。
- 增加引用：赋值绑定、传参、放入容器、作为属性。
- 减少引用：名字被重新绑定、`del`、离开作用域、容器删除元素。
- **计数归零的瞬间立即释放**（确定性析构，`__del__` 此刻触发）——这与 Java/Go 的追踪式 GC 的"不确定何时回收"是根本差异。

### 循环引用问题

```python
a = []
b = []
a.append(b)
b.append(a)
del a, b   # 两个对象互相引用，计数永远不为 0
```

引用计数解决不了环，所以 CPython 另有 **分代 GC（`gc` 模块）**：

- 只跟踪**容器对象**（list、dict、tuple、自定义类实例等能持有引用的对象）；纯标量不可能成环，不参与。
- 三代（generation 0/1/2），对象在 GC 扫描中存活则晋升；第 0 代分配次数超过阈值（默认 700 次分配-释放差）触发扫描。
- 核心算法：对每个容器做"计数扣减"——扣掉容器内部互相引用的部分，剩下的非零计数说明有外部引用可达；计数为 0 的孤岛即为环垃圾。

**实践认知：**

- 大多数对象靠引用计数立刻释放，`gc` 只兜底环。
- `__del__` + 循环引用在 Python < 3.4 会导致无法回收；3.4 起（PEP 442）可以回收，但 `__del__` 仍是坏味道，请用 `weakref` 或上下文管理器。
- `gc.disable()` 在某些服务（如 prefork Web 服务器）中是真实存在的性能优化手段。

---

## 4. 内存管理的细节：小整数池、驻留、pymalloc

CPython 为对抗"万物皆对象"带来的分配开销，做了大量缓存：

| 机制 | 内容 | 可观测后果 |
|---|---|---|
| 小整数池 | `[-5, 256]` 预创建，全程复用 | `257 is 257` 可能 False |
| 字符串驻留（intern） | 短标识符样式字符串自动驻留；`sys.intern()` 手动 | `a = "hello"; b = "hello"; a is b` 常为 True |
| free list | int/float/list/dict 等类型的空闲对象池，释放不还给 OS | 内存占用"涨上去下不来" |
| pymalloc | ≤512 字节的小对象走私有分配器（arena→pool→block 三级） | 减少 malloc 调用与碎片 |
| 空元组/单例 | `()`、`None`、`True`、`False`、`...` 全局唯一 | `() is ()` 恒 True |

**认知要点：** 这些优化让你偶尔会观察到"对象复用"，但它们全是**实现细节而非语言承诺**。写代码时唯一正确的姿势：用 `==` 比值，用 `is` 只比单例。

---

## 5. GIL：为什么存在、意味着什么、正在发生什么变化

### 它是什么

GIL（Global Interpreter Lock）是 CPython 解释器级别的一把互斥锁：**同一时刻只有一个线程在执行 Python 字节码**（持有 GIL）。

### 为什么存在

因为引用计数不是线程安全的。若不加锁，两个线程同时增减 `ob_refcnt` 会产生竞态，导致内存泄漏或崩溃。给每个对象配锁代价巨大且极易死锁，于是 Guido 选择了"一把大锁保平安"——这是 1992 年单核时代的合理取舍，并深刻塑造了 C 扩展生态（NumPy 等大量扩展依赖 GIL 简化内存管理假设）。

### 实际影响

- **CPU 密集**多线程：多线程≈单线程甚至更慢（锁竞争 + 切换开销）。
- **IO 密集**多线程：依然有效——线程在阻塞 IO（网络、文件、`time.sleep`）时会**释放 GIL**。
- 切换机制：3.2 起，线程默认每 **5ms**（`sys.setswitchinterval`）强制让出；不再按字节码条数。
- C 扩展（如 NumPy 的重计算）也可以主动释放 GIL，这就是 NumPy 多线程能提速的原因。

### 绕行方案

- 多进程：`multiprocessing`、`concurrent.futures.ProcessPoolExecutor`（3.13+ 还有 `InterpreterPoolExecutor`，子解释器各有独立 GIL）。
- C 扩展释放 GIL。
- **PEP 703（no-GIL / free-threading）**：Python 3.13 引入实验性无 GIL 构建（`python3.13t`），用偏向锁 + 延迟引用计数 + 分代 GC 改造替换 GIL。这是 Python 并发史上最大的变革，但目前仍是实验特性，C 扩展生态需要跟进。

**认知要点：** GIL 保护的是**解释器内部状态**，不是**你的数据**。`x += 1` 在多线程下依然不是原子的（它是 LOAD/ADD/STORE 多条字节码，3.11 前还可能中途切换线程），该加的锁一把不能少。

---

## 6. 作用域与闭包：LEGB、cell、晚期绑定

### LEGB

名字解析顺序：**Local → Enclosing → Global → Builtins**。关键：这个规则在**编译期**就决定每个名字的归属（通过符号表），运行期只是按既定槽位查。

### 闭包的实现：cell 对象

```python
def outer():
    x = 10
    def inner():
        return x   # 引用外层局部变量
    return inner

f = outer()
print(f())            # 10 —— outer 已返回，x 还活着
print(f.__closure__)  # (<cell at ...: int object at ...>,)
```

原理：当编译器发现内层函数引用了外层局部变量，就把该变量从"栈帧局部"提升为堆上的 **cell 对象**（一个盒子），内外层函数各自持有一个指向同一个 cell 的指针。外层栈帧销毁后，cell 仍存活于堆上。这就是闭包 = "函数对象 + 其引用的 cell 元组（`__closure__`）"。

### 晚期绑定陷阱

```python
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]   # [2, 2, 2]，不是 [0, 1, 2]
```

三个 lambda 共享**同一个 cell**（循环变量 `i` 的 cell），循环结束后 cell 里的值是 2。闭包捕获的是**变量（cell）本身，不是当时的值**。

修复——把值"冻结"到默认参数里（默认参数在 def 时求值）：

```python
funcs = [lambda i=i: i for i in range(3)]
```

---

## 7. `global` 与 `nonlocal`：不是声明，是"绑定指令"

二者都不是"引入一个变量"，而是**改变编译器对名字的归类**：

- `global x`：告诉编译器，本函数内对 `x` 的所有绑定/读取都指向**模块全局命名空间**。
- `nonlocal x`：指向**最近的外层函数作用域**中已存在的 `x`（找不到定义则 `SyntaxError`；不能穿透到全局）。

```python
def counter():
    n = 0
    def inc():
        nonlocal n   # 没有这行，n += 1 会报 UnboundLocalError
        n += 1
        return n
    return inc
```

为什么 `n += 1` 需要 `nonlocal` 而 `lst.append(1)` 不需要？因为 `+=` 对 int 是**重新绑定**（int 不可变），`append` 是**原地修改**（不涉及绑定）。再次印证：Python 语义的核心分界线是"绑定 vs 修改"。

---

## 8. `is` 与 `==`、`del` 的真实语义

- `a is b` ⟺ `id(a) == id(b)`，比较身份，不可重载。
- `a == b` 调用 `a.__eq__(b)`；`a != b` 默认取反（3.x 自动），`__eq__` 未定义时退化为身份比较。
- 定义 `__eq__` 而未定义 `__hash__`，类会被置为不可哈希（`__hash__ = None`）——因为"相等则哈希必等"是 dict/set 正确性的基石。
- `del name`：**解除绑定**（引用计数 -1），不是"销毁对象"。对象何时销毁取决于是否还有其他引用。`del lst[0]` 是容器的 `__delitem__`，与解除名字绑定是两码事，只是共享了关键字。

---

## 9. 函数即对象：默认参数陷阱与 `*args/**kwargs`

### 默认参数在 def 时求值一次

```python
def f(x, acc=[]):
    acc.append(x)
    return acc

f(1)  # [1]
f(2)  # [1, 2] —— 同一个 list！
```

`def` 是**执行语句**：执行时创建函数对象，默认参数**求值一次**并存入 `f.__defaults__`。后续所有调用共享该对象。

正确写法：

```python
def f(x, acc=None):
    if acc is None:
        acc = []
```

这个"陷阱"反过来也是一种技巧：用默认参数做缓存/寄存局部状态（早年装饰器、`functools.lru_cache` 之前的记忆化写法）。

### 函数对象的属性

```python
def g(a, b=1, *args, **kwargs): ...
g.__defaults__   # (1,)
g.__code__       # 代码对象：字节码、常量、变量名、栈大小……
g.__globals__    # 定义处的全局命名空间（字典）
g.__closure__    # 闭包 cell 或 None
```

装饰器、热重载、调试器、序列化框架，全都建立在这些属性之上。

---

## 10. 装饰器：语法糖之下的真相

```python
@deco
def f(): ...
```

**精确等价于：**

```python
def f(): ...
f = deco(f)
```

三个推论，全是高频坑：

1. **装饰发生在 def 执行时（import 时），不是调用时。** 模块一被导入，装饰器就跑了——所以装饰器里写副作用（注册、打日志）要谨慎 import 顺序。
2. **装饰器替换的是名字绑定。** 类中的方法被装饰后，原函数只能通过闭包或注册表找回。
3. **叠加顺序自下而上：** `@a @b def f` ⟺ `f = a(b(f))`，离函数最近的先应用。

带参装饰器其实是"装饰器工厂"：`@deco(arg)` ⟺ `f = deco(arg)(f)`，多一层调用。

`functools.wraps` 不是形式主义——它把 `__name__`、`__doc__`、`__module__`、`__wrapped__` 从原函数复制到包装函数，否则调试、自省、`help()`、以及依赖签名的框架（Flask 路由名、pytest）都会出错。

---

## 11. 迭代协议与生成器：`yield` 的本质是"可暂停的栈帧"

### 迭代协议

```python
for x in it:
```
展开为：`iter(it)` 调用 `__iter__` 拿到迭代器 → 反复 `next()` 调用 `__next__` → 捕获 `StopIteration` 结束循环。

可迭代 ≠ 迭代器：`list` 可迭代（`__iter__` 返回新迭代器），`list_iterator` 是迭代器（`__iter__` 返回自身 + `__next__`）。

### 生成器的实现

普通函数调用时，CPython 创建一个**栈帧（frame）**执行，返回即销毁。生成器函数被调用时**不执行**，只返回一个生成器对象——它内部持有自己的栈帧。每次 `next()`：

1. 恢复该栈帧，从上次 `yield` 处继续执行；
2. 遇到下一个 `yield`，把值抛给调用者，**栈帧整体冻结保留**；
3. 函数 return / 结束 → 抛 `StopIteration`。

因为栈帧存活在堆上，生成器就实现了"用同步代码写惰性流"。内存友好的根源：**任何时刻只在内存中保留一个元素的处理状态**。

### `send()` 与双向通信

```python
def coro():
    while True:
        x = yield        # 暂停点也是表达式：x 接收 send 进来的值
        print("got", x)

c = coro()
next(c)          # 预激（prime）：推进到第一个 yield
c.send(42)       # got 42
```

`yield` 是**表达式**而非语句——这是协程的支点。`gen.send(v)` 把 `v` 注入暂停点作为 `yield` 表达式的值，生成器从此处继续跑。`throw()` 则是在暂停点**抛入异常**。

---

## 12. `yield from` 与协程：生成器如何长成 async/await

`yield from sub` 不是"逐个转发"的语法糖，它做了四件事：

1. 委托迭代：把 `next/send/throw/close` **透明转发**给子生成器；
2. 双向管道：调用方的 `send()` 直达子生成器；
3. 捕获返回值：子生成器 `return v` 时，`yield from` 表达式的值就是 `v`（`StopIteration.value`）；
4. 异常传播：子生成器的异常原样向上抛。

```python
def sub():
    yield 1
    return "done"

def main():
    result = yield from sub()
    print(result)   # "done"
```

### 从生成器到 async/await

历史脉络：

- Python 2.5：`send/throw/close` → 生成器可当协程用。
- Python 3.3（PEP 380）：`yield from` → 协程可以**嵌套调用**并形成调用栈。
- Python 3.4：`@asyncio.coroutine` + `yield from` = 第一个可用的 async 框架。
- Python 3.5（PEP 492）：`async def` / `await` 成为正式语法。**`await` 在实现层面就是 `yield from` 的特化**（只允许 await 实现了 `__await__` 的对象），async 函数本质仍是"由事件循环驱动 send 的生成器"。
- `asyncio` 事件循环的角色：维护任务队列，任务在 `await` 处让出（等价于 yield），IO 就绪后再 send 回去。**单线程并发的秘密就是：所有阻塞都发生在事件循环的一个点上。**

**认知要点：** `async` 不产生并行，它产生的是**协作式并发**。任何在协程里调用的阻塞库（`requests`、`time.sleep`）都会冻结整个事件循环——必须换成 `aiohttp`、`asyncio.sleep`。

---

## 13. 描述符协议：属性访问背后的总开关

任何定义了 `__get__` / `__set__` / `__delete__` 之一的对象就是**描述符**。属性访问 `obj.attr` 的真实解析顺序（`object.__getattribute__`）：

1. 在 `type(obj).__mro__` 中查找 `attr`；
2. 若找到且是**数据描述符**（定义了 `__set__` 或 `__delete__`）→ 调用其 `__get__`，结束；
3. 查 `obj.__dict__`（实例字典）；
4. 若类中找到的是**非数据描述符**（只有 `__get__`）→ 调用之；
5. 否则返回类属性，再找不到 → `__getattr__`（如果定义了）→ `AttributeError`。

这个协议是 Python 半壁江山的地基：

| 你用的东西 | 实际是 |
|---|---|
| `@property` | 数据描述符 |
| 方法绑定 `obj.method` | 函数是**非数据描述符**，`__get__` 返回 bound method（把 `self` 塞进去） |
| `@staticmethod` / `@classmethod` | 改变了 `__get__` 行为的描述符 |
| ORM 的字段（`CharField` 等） | 描述符 |
| `__getattr__` / `__setattr__` | 拦截兜底 |

**方法绑定的真相：**

```python
class A:
    def f(self): ...

A.f        # 函数对象（3.x 起就是普通函数）
a = A()
a.f        # <bound method A.f of <A object>> —— A.f.__get__(a, A) 的产物
```

所以 `a.f()` ⟺ `A.f(a)`——`self` 没有任何魔法，就是描述符协议塞进来的第一个参数。

---

## 14. 元类与 `type`：类也是对象

`class` 语句的执行过程：

```python
class Foo(Base, metaclass=Meta):
    x = 1
```

大致等价于：

```python
namespace = Meta.__prepare__("Foo", (Base,))   # 准备命名空间（默认普通 dict）
exec(class_body, namespace)                     # 执行类体，填充命名空间
Foo = Meta("Foo", (Base,), namespace)           # 调用元类创建类对象
```

**`type` 的三个身份：**

1. `type(obj)` —— 查询类型；
2. `type(name, bases, ns)` —— **动态创建类**（ORM、序列化框架、dataclass 底层的核心手段）；
3. 所有类的默认元类，`type` 自己的元类也是 `type`（自举闭环）。

元类的真实用途是**拦截类的创建**：在 `Meta.__new__/__init__` 里检查/修改命名空间、自动注册子类、注入方法。Django Model 的字段收集、SQLAlchemy 的声明式映射，都是元类（或其后继方案）的杰作。

现代替代方案（多数场景不再需要写元类）：

- `__init_subclass__(cls, **kwargs)`：父类定义，**子类创建时**自动回调——注册、校验的最常见需求用它就够。
- `__set_name__(self, owner, name)`：描述符在类创建完成后被告知其属性名——解决了描述符过去"不知道自己叫什么"的难题（ORM 字段不再需要显式传名字）。

---

## 15. MRO 与 C3 线性化

### 为什么需要 MRO？

多重继承时，`D` 同时继承 `B` 和 `C`，`B` 和 `C` 又都继承 `A`。调用 `D()` 的某个方法时，Python 必须回答一个顺序问题：**按什么顺序查找？**

```
      A
     / \
    B   C
     \ /
      D
```

这个查找顺序就是 **MRO（Method Resolution Order）**。Python 2.3+ 使用 **C3 线性化算法**来计算它。

---

### C3 的三条原则（用人话说）

| # | 原则 | 人话 |
|---|------|------|
| 1 | **子类优先** | 子类永远排在父类前面 |
| 2 | **保持声明顺序** | `class D(B, C)` 中 B 排在 C 前面 |
| 3 | **单调性** | 如果 A 在 B 前面出现在某处，最终结果中 A 也一定在 B 前面 |

三条原则**必须同时满足**，满足不了就直接报错：

```python
class A: pass
class B(A): pass
class C(A, B): pass   # TypeError: Cannot create a consistent MRO
#                            C 说 "A 在 B 前面"，但 B 说 "我在 A 后面" → 矛盾 → 报错
```

---

### MRO 怎么算：一个完整的手算过程

看这个继承结构：

```python
class A:
    def greet(self): print("A")

class B(A):
    def greet(self): print("B")

class C(A):
    def greet(self): print("C")

class D(B, C):
    pass
```

C3 的计算思路（简化版）：

1. **从最底层 D 开始**，把 D 放进结果列表 → `[D]`
2. **D 的直接父类是 B、C**（按声明顺序），把 B 加入 → `[D, B]`
3. **B 的父类是 A**，但 A 也是 C 的父类——A 还没处理完，先跳过，继续处理 C → `[D, B, C]`
4. **最后加入 A** → `[D, B, C, A]`

用 `__mro__` 验证：

```python
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

再看一个更复杂的（菱形继承的复杂版）：

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
class E(D):    pass
class F(E, B): pass
```

手动推导 F 的 MRO：

1. `[F]`
2. F 的父类是 E、B → `[F, E]`（B 等一等，E 的父类还没处理完）
3. E 的父类是 D → `[F, E, D]`
4. D 的父类是 B、C → `[F, E, D, B]`（C 等一等）
5. B 的父类是 A → `[F, E, D, B, C]`（C 还没处理）
6. C 的父类是 A → `[F, E, D, B, C, A]`
7. 最后 object → `[F, E, D, B, C, A, object]`

```python
print(F.__mro__)
# (<class 'F'>, <class 'E'>, <class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

---

### `super()` 的真正含义：沿 MRO 找下一个，不是"调用父类"

这是最容易搞混的地方。`super()` 的行为：

> **在当前类的 MRO 中，从当前位置的下一个类开始查找方法。**

用上面的 A/B/C/D 结构举例：

```python
class A:
    def greet(self):
        print("A")

class B(A):
    def greet(self):
        print("B")
        super().greet()  # 不是 "调 A.greet"，而是 "沿 MRO 找下一个"

class C(A):
    def greet(self):
        print("C")
        super().greet()

class D(B, C):
    pass
```

调用 `D().greet()`，执行过程：

```
MRO: D → B → C → A → object

1. D 没定义 greet → 跳过
2. 找到 B.greet() → 打印 "B"
3. B 的 super() → 沿 MRO 找 B 的下一个 → 是 C → 打印 "C"
4. C 的 super() → 沿 MRO 找 C 的下一个 → 是 A → 打印 "A"
```

输出：`B → C → A`

**关键理解**：B 里的 `super()` 不是跳回 A，而是沿 MRO 往下走一步。所以 B 的 `super()` 调到了 **C**，而不是 A。

---

### 协作式继承的规则

`super()` 能正确工作，前提是**每一层都调用 `super()`**：

```python
class A:
    def process(self):
        print("A")

class B(A):
    def process(self):
        print("B")
        super().process()   # ✅ 必须调

class C(A):
    def process(self):
        print("C")
        super().process()   # ✅ 必须调

class D(B, C):
    def process(self):
        print("D")
        super().process()   # ✅ 必须调
```

如果某一层**没调 `super()`**，链就断了——后面的类永远不会被执行。

无参 `super()` 的魔法来自编译器自动注入的 `__class__` cell（隐式引用当前类和 `self`）。

---

## 16. `__slots__`：省内存的代价

普通实例的属性存在 `__dict__`（哈希表，内存大）。`__slots__ = ("x", "y")` 告诉 CPython：不要 `__dict__`，把属性存在**类级别预生成的描述符 + 实例的定长槽位数组**里。

效果：单实例内存常省 40–60%，属性访问略快，且禁止随意添加新属性。

代价与坑：

- **继承不继承"无 dict"特性**：子类不声明 `__slots__`，dict 又回来了（且 slots 与 dict 并存，内存反而更贵）。
- 失去弱引用（除非加 `"__weakref__"`）。
- 与多继承、某些描述符、`functools.cached_property`（依赖实例 dict）存在交互问题。
- slots 里的名字是**类级描述符**，类属性同名会冲突。

适用场景：海量小对象（百万级实例）。除此之外优先 `@dataclass(slots=True)`（3.10+ 一行搞定）。

---

## 17. 上下文管理器与 `with` 协议

```python
with EXPR as x:
    BODY
```

展开：

```python
mgr = EXPR
exit_ = type(mgr).__exit__
x = type(mgr).__enter__(mgr)     # __enter__ 的返回值绑定给 as 变量
try:
    BODY
except BaseException as e:
    if not exit_(mgr, type(e), e, e.__traceback__):   # 返回真值 = 吞掉异常
        raise
else:
    exit_(mgr, None, None, None)
```

关键认知：

- `as x` 绑定的是 `__enter__` 的**返回值**，不是 `EXPR` 本身（`open(...) as f` 中两者恰好相同，纯属巧合设计）。
- `__exit__` 返回真值会**抑制异常**——`contextlib.suppress` 就靠这个。
- `contextlib.contextmanager` 用生成器实现同一协议：`yield` 之前是 `__enter__`，之后是 `__exit__`，`yield` 处的值即 as 目标；异常通过 `throw()` 注入 `yield` 点——这就是为什么用生成器写 CM 时要在 `yield` 外套 `try/finally`。

---

## 18. 异常机制：异常是对象，栈帧是代价

- 异常实例携带 `__traceback__`（帧链）、`__context__`（隐式链）、`__cause__`（`raise ... from ...` 显式链）、`__notes__`（3.11+）。
- `raise NewErr() from e` 把 `__cause__` 设为 e，打印时显示 "The above exception was the direct cause..."——**异常转换时的正确姿势**，信息链完整。
- `except` 子句中发生的异常自动把当前异常链入 `__context__`（"During handling of the above exception..."）。
- `finally` 的语义陷阱：
  - `finally` 中的 `return` 会**吞掉** try 里未处理的异常（3.14 起对此发 SyntaxWarning）。
  - `finally` 总是执行，即使 try 里有 `return`/`break`/`continue`。
- `else` 子句：只在无异常时执行，用于收窄 try 的范围——**try 块里只放真正可能抛错的那一行**是异常风格的最佳实践。
- 成本认知：3.11+ 采用"零成本异常"——不抛异常时几乎零开销（查表），抛异常时才付出构建 traceback 的代价。所以 Python 的 EAFP 风格（先干了再说，错了再捕获）在性能上是站得住的。

---

## 19. import 系统：模块是单例，循环导入为何出错

### 执行模型

`import foo` 的完整流程：

1. 查 `sys.modules`——命中则直接返回缓存（**模块是进程级单例**）；
2. 否则按 `sys.meta_path` 的 finder 链找到 spec（文件路径）；
3. 创建模块对象，**先放入 `sys.modules`**（这一步是理解循环导入的钥匙）；
4. `exec` 模块顶层代码，填充模块的 `__dict__`。

### 循环导入为什么有时报错有时不

```python
# a.py
import b
def fa(): b.fb()

# b.py
import a
def fb(): a.fa()
```

这种"只 import 模块、运行期才用"的循环引用**没问题**：因为 a 执行到一半时已在 `sys.modules` 里，b `import a` 拿到的是半成品的 a 模块对象，但只要运行期才访问属性，届时 a 已执行完毕。

而 `from a import fa` 的循环版本会炸：`from X import Y` 在 import 时就取属性 `Y`，而半成品的 a 里 `fa` 还没定义。

**认知要点：** import 就是"执行一个文件并把结果缓存进 `sys.modules`"。`importlib.reload()` 是原地重新执行，旧引用不会自动更新——这是热重载总是半吊子的原因。

### 其他细节

- `__pycache__`：编译产物按版本缓存（`foo.cpython-311.pyc`），import 慢主要是磁盘 IO 和顶层代码执行，不是编译。
- 包即模块：`__init__.py` 是包的模块体，`import pkg.sub` 会先执行 `pkg/__init__.py`——在 `__init__` 里写重逻辑会拖慢所有子模块导入。
- 顶层副作用（连接数据库、起线程）是 import 系统的万恶之源：它让 import 顺序变得有意义且脆弱。

---

## 20. 字节码与执行模型：`dis` 看世界

```python
import dis
def f(a, b):
    return a + b
dis.dis(f)
```

```
LOAD_FAST        a
LOAD_FAST        b
BINARY_OP        + 
RETURN_VALUE
```

CPython 执行流程：源码 → AST → **字节码**（存于 code 对象）→ 栈式虚拟机逐条执行。

值得知道的执行细节：

- **栈式 VM**：没有寄存器，操作数在求值栈上推入弹出。每条字节码都是 C 层的一个巨大 switch（3.11+ 是 computed goto + specializing）。
- **自适应解释器（PEP 659，3.11+）**：热点字节码在运行中被替换为特化版本（如 `BINARY_OP_ADD_INT`），带来 3.11 的 10–60% 提速。3.13 又加了复制补丁（copy-and-patch）JIT 的实验构建。
- **函数调用很贵**：每次调用要分配 frame 对象、拷贝参数、建立局部变量槽。这就是为什么 Python 里"少一层函数调用"是真实有效的优化，而内联常由人类手动完成。
- `LOAD_FAST`（局部变量，数组索引）比 `LOAD_GLOBAL`（字典查找）快——把循环里反复用的全局函数绑成局部名（`append = lst.append`）是有依据的微优化。
- 递归限制 `sys.getrecursionlimit()`（默认 1000）本质是防止 C 栈溢出；3.12+ 纯 Python 递归基本不吃 C 栈，限制更多是一种保护。

---

## 21. 拷贝语义：`copy` 与 `deepcopy`

```python
import copy
a = [[1, 2], [3, 4]]
b = copy.copy(a)       # 浅拷贝：新容器，共享元素
c = copy.deepcopy(a)   # 深拷贝：递归复制整张对象图

b[0].append(9)
a     # [[1, 2, 9], [3, 4]] —— 共享内层 list
c     # [[1, 2], [3, 4]]   —— 不受影响
```

- 浅拷贝 = 新容器 + 原元素的**引用**。`list(a)`、`a[:]`、`dict(a)` 都是浅拷贝。
- 深拷贝用 **memo 字典**记录已拷贝对象：保证共享子对象在副本中依然共享（拓扑保持），并能处理循环引用。
- 自定义类可实现 `__copy__` / `__deepcopy__` 控制行为。
- 不可变对象的拷贝通常被优化为返回原对象（反正改不了，拷了也白拷）。

---

## 22. 常见认知误区速查表

| 误区 | 真相 |
|---|---|
| `is` 是"更严格的等于" | `is` 是身份比较，与相等无关 |
| Python 传参是"值传递/引用传递" | 是"对象引用传递"：共享对象，但重绑定不影响调用方 |
| 多线程能加速 CPU 密集任务 | CPython 有 GIL；用多进程或 3.13t |
| `del x` 销毁对象 | 只解除一个绑定 |
| 默认参数每次调用重新创建 | def 时求值一次，全程共享 |
| 闭包捕获变量的值 | 捕获 cell（变量本体），是晚期绑定 |
| `super()` 调用父类 | 沿 MRO 找下一个 |
| `with open() as f` 中 f 是 open 的返回值 | f 是 `__enter__` 的返回值（此处恰好相同） |
| `async` 带来并行 | 带来协作式并发，阻塞调用照样冻住事件循环 |
| `__slots__` 声明后子类也省内存 | 子类需各自声明 |
| 异常很贵，要少用 | 3.11+ 零成本异常：不抛不花钱，EAFP 放心用 |
| 元类是日常工具 | 90% 场景 `__init_subclass__` / 装饰器即可替代 |
| 生成器是"返回列表的函数" | 是持有冻结栈帧的迭代器，惰性、单次、不可重放 |

---

## 结语：一条主线

Python 的几乎所有"反直觉"都能归结到三条基本原理：

1. **一切皆对象，赋值皆绑定。** 可变/不可变、传参、del、拷贝，全是这一条的推论。
2. **协议优于类型。** 迭代、`with`、属性访问、比较、`await`……全是"定义魔术方法即获得语法支持"的描述符/协议驱动。Python 的可扩展性来自这里。
3. **运行时是透明的。** 函数有 `__code__`、闭包有 `__closure__`、类有 `__mro__`、模块在 `sys.modules`——解释器内部结构几乎全部暴露为 Python 层对象。学会用 `dis`、`inspect`、`gc` 去观察它们，是从"会用"到"懂"的分水岭。
