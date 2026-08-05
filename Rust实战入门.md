# Rust 实战入门：给"只想写 GUI 和脚本"的开发者

> 与《[Rust 深度剖析](./Rust深度剖析.md)》互为补充：那份讲"为什么"，这份讲"怎么用"。本文的目标读者是**应用开发者**——不写编译器、不写操作系统内核，只想用 Rust 快速做出一个 GUI 软件、一个命令行小工具、一个自动化脚本。
>
> 原则：**够用就好**。文中标注了"必须掌握 / 尽量掌握 / 现在不用学"三档，帮助你只付出最少的学习成本。

---

## 目录

1. [先破除两个误区](#1-先破除两个误区)
2. [你需要掌握什么：知识清单](#2-你需要掌握什么知识清单)
3. [环境与项目管理：Cargo](#3-环境与项目管理cargo)
4. [语法最小集：写代码够用即可](#4-语法最小集写代码够用即可)
5. [写脚本：把 Rust 当 Python 用](#5-写脚本把-rust-当-python-用)
6. [开发 GUI：以 egui/eframe 为例](#6-开发-gui以-egueframe-为例)
7. [GUI 进阶：线程通信与文件对话框](#7-gui-进阶线程通信与文件对话框)
8. [其他 GUI 框架：何时选它们](#8-其他-gui-框架何时选它们)
9. [常用 crate 速查表](#9-常用-crate-速查表)
10. [高频编译错误与解法速查](#10-高频编译错误与解法速查)
11. [完整综合示例：带 GUI 的批量重命名工具](#11-完整综合示例带-gui-的批量重命名工具)
12. [务实学习路线](#12-务实学习路线)

---

## 1. 先破除两个误区

**误区一：必须精通所有权/生命周期才能写 Rust。**

真相：应用开发中 90% 的代码根本不需要手写生命周期标注（编译器会自己推断）。你只需要掌握 3 条规则 + 6 个高频报错的解法（见第 10 节）。Rust 的借用检查器像一个严格的 review，但它的报错信息几乎每次都直接告诉你改法。

**误区二：Rust 不适合写 GUI 和脚本。**

真相：截至 2026 年，纯 Rust 原生渲染的 GUI 框架中，`egui/eframe` 的 crates.io 累计下载量超过 3700 万次，断层领先（是其余纯 Rust 框架总和的数倍），由 Rerun 公司全职维护并用于生产级可视化产品；命令行工具方向更是 Rust 的统治区（`ripgrep`、`bat`、`fd` 等）。Rust 写脚本的短板是**编译慢**（一次 `cargo build` 数秒到数十秒），优势是**单文件发布、启动零依赖、性能接近 C**——适合"要分发给别人用的小工具"。

> 何时用 Rust 写脚本：需要性能、需要单文件分发、复用现有 Rust 库。
> 何时别用：纯快速原型、胶水脚本、频繁改逻辑的探索性代码 → 用 Python/Shell 更合适。

---

## 2. 你需要掌握什么：知识清单

| 优先级 | 内容 | 说明 |
|---|---|---|
| 🟢 必须掌握 | 变量/类型/函数、`struct`、`enum`、`match`、`if let` | 任何语言的通用基础 |
| 🟢 必须掌握 | `Option<T>` / `Result<T, E>` + `?` 操作符 | Rust 版的"null 和异常"，每天用无数次 |
| 🟢 必须掌握 | `String` / `&str` / `Vec<T>` / `HashMap` 的常用方法 | 见 §4.4 |
| 🟢 必须掌握 | 借用三规则 + `move` 闭包 | 见 §4.6，GUI 里闭包满天飞 |
| 🟢 必须掌握 | 错误处理：`main` 返回 `Result`、`?`、`unwrap/expect` 的取舍 | 见 §4.5 |
| 🟢 必须掌握 | Cargo：建项目、加依赖、跑测试、编译发布 | 见 §3 |
| 🟡 尽量掌握 | trait 的**使用**（不用会定义）：`Debug`/`Clone`/`Default` 派生、`impl Trait` | 90% 时候只是 `#[derive(...)]` |
| 🟡 尽量掌握 | 迭代器链 `.iter().filter().map().collect()` | 数据处理的主武器 |
| 🟡 尽量掌握 | `async/await` 基础（配 tokio） | 写网络请求时会遇到，见 §5.7 |
| 🟡 尽量掌握 | 闭包与 `FnOnce/FnMut/Fn` 的直觉 | 不必背定义，出错时能看懂报错即可 |
| 🔴 现在不用学 | `unsafe` | 写应用几乎用不到 |
| 🔴 现在不用学 | 过程宏（proc macro） | 你只会**用** `#[derive(Serialize)]`，不需要会写 |
| 🔴 现在不用学 | `Pin` 原理、`Future` 状态机内部 | 那是《深度剖析》的事 |
| 🔴 现在不用学 | 生命周期标注的完整规则 | 见 §4.6 的"省略规则"，编译器帮你写 |
| 🔴 现在不用学 | 型变、对象安全、vtable 细节 | 碰到 `dyn` 报错再查 |
| 🔴 现在不用学 | `Cell`/`RefCell` | 多线程 UI 场景用 `Arc<Mutex>` 模板即可 |

一句话总结：**普通应用开发者只需要"会读借用检查器的报错"，不需要"会写复杂的生命周期"。**

---

## 3. 环境与项目管理：Cargo

### 3.1 安装与常用命令

```sh
# 安装 rustup（工具链管理器）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 常用命令
cargo new my_app          # 新建二进制项目
cargo new my_lib --lib    # 新建库项目
cd my_app
cargo add eframe          # 添加依赖（会写入 Cargo.toml）
cargo run                 # 编译并运行
cargo build --release     # 发布构建（优化 + 无调试信息，体积小、速度快）
cargo check               # 只检查不生成二进制，最快反馈
cargo test                # 跑测试
cargo fmt                 # 代码格式化（rustfmt，社区统一风格）
cargo clippy              # 更严格的 lint，建议每写完一段跑一次
```

> 新手最舒服的工作流：开一个终端跑 `cargo run`，每改完一个函数按一次保存。

### 3.2 项目结构

```
my_app/
├── Cargo.toml      # 项目元信息 + 依赖清单
├── Cargo.lock      # 锁定精确版本（自动生成，提交到 git）
└── src/
    └── main.rs     # 入口
```

`Cargo.toml` 示例：

```toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2021"            # 版本契约：2021/2024

[dependencies]
eframe = "0.31"             # "0.31" 表示 >=0.31, <0.32
serde = { version = "1", features = ["derive"] }
```

> **习惯**：`cargo add 包名` 永远比手改 `Cargo.toml` 稳妥，它会自动挑最新兼容版本。

### 3.3 模块：多文件组织

文件一多就要拆模块。规则极简：`src/foo.rs` 里写了 `pub fn bar()`，在 `main.rs` 里 `mod foo;` 就能用 `foo::bar()`：

```rust
// src/math_utils.rs
pub fn double(x: i32) -> i32 { x * 2 }

// src/main.rs
mod math_utils;              // 声明模块，编译器去找 math_utils.rs

fn main() {
    println!("{}", math_utils::double(21)); // 42
}
```

---

## 4. 语法最小集：写代码够用即可

### 4.1 变量、函数、结构体、枚举

```rust
#[derive(Debug, Clone)]        // derive：免费获得 Debug 打印和 Clone 复制能力
struct Task {
    id: u32,
    title: String,
    done: bool,
}

enum Status {                  // 枚举可携带数据，这是 Rust 表达"或"的方式
    Pending,
    Done,
    Failed(String),            // 携带错误信息
}

fn add(a: i32, b: i32) -> i32 { a + b }   // 最后一个表达式是返回值，不用 return

fn main() {
    let mut t = Task { id: 1, title: "写文档".into(), done: false };  // .into() 把 &str 转 String
    t.done = true;

    let s = Status::Failed("磁盘满".to_string());
    match s {
        Status::Pending => println!("待办"),
        Status::Done => println!("完成"),
        Status::Failed(msg) => println!("失败：{msg}"),  // 解构出携带的数据
    }
    println!("{:?}", t);       // {:?} 打印 Debug
}
```

### 4.2 `Option` 与 `Result`：没有 null，没有异常

```rust
fn find_user(id: u32) -> Option<String> {      // Option: 可能有，可能没有
    if id == 1 { Some("Alice".to_string()) } else { None }
}

fn read_file(path: &str) -> Result<String, std::io::Error> {  // Result: 可能成功，可能失败
    std::fs::read_to_string(path)
}

fn main() {
    // Option 的两种消费方式
    match find_user(1) {
        Some(name) => println!("{name}"),
        None => println!("找不到"),
    }
    if let Some(name) = find_user(2) {          // 只关心 Some 分支时的简写
        println!("{name}");
    }

    // `?` 操作符：失败就提前返回，成功就取出值（只能在返回 Result/Option 的函数里用）
    // 配合 main 返回 Result，整个脚本的主流程可以一路 `?` 下去
    let content = read_file("Cargo.toml")?;     // 失败：打印错误并退出
    println!("{}", content.lines().count());
}
```

> `unwrap()` 相当于"我确信不会失败，失败就 panic"；`expect("说明")` 加上原因。开发期随便用，发布前的边界路径换成 `?` 或 `match`。

### 4.3 集合：`Vec` / `HashMap` / `String`

```rust
use std::collections::HashMap;

fn main() {
    // Vec：动态数组
    let mut v = vec![1, 2, 3];
    v.push(4);
    let first = v.first();                 // Option<&i32>
    let sum: i32 = v.iter().sum();         // 10

    // HashMap：键值对
    let mut scores = HashMap::new();
    scores.insert("rust".to_string(), 99);
    *scores.entry("go".to_string()).or_insert(0) += 1;   // 不存在则插入 0，再 +1

    // String 常用操作
    let mut s = String::from("hello");
    s.push(' '); s.push_str("world");
    let upper = s.to_uppercase();
    let replaced = s.replace("world", "rust");
    let words: Vec<&str> = s.split(' ').collect();
    println!("{}", s.contains("rust"));    // false
}
```

### 4.4 迭代器链：数据处理的主武器

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6];

    let result: Vec<i32> = numbers
        .iter()            // 惰性迭代器
        .filter(|n| *n % 2 == 0)   // 过滤：偶数
        .map(|n| n * 10)           // 变换
        .collect();        // 收集回 Vec
    assert_eq!(result, vec![20, 40, 60]);

    // 常见组合
    let count = numbers.iter().filter(|n| **n > 3).count();
    let max = numbers.iter().max().unwrap();
    let joined = numbers.iter().map(|n| n.to_string()).collect::<Vec<_>>().join(", ");
}
```

> `Vec::iter()` 产出 `&i32`，所以闭包里是 `*n`（解引用）；想直接拿所有权用 `.into_iter()`。

### 4.5 错误处理：统一模板

```rust
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {   // Box<dyn Error> = "任意错误类型"，脚本/工具类最省事
    let path = std::env::args().nth(1).ok_or("请传入文件路径参数")?;  // Option → Result 的快捷转换
    let content = std::fs::read_to_string(&path)?;   // io::Error 自动转成 Box<dyn Error>
    let n: usize = content.trim().parse()?;          // 解析失败也会自动转
    println!("文件包含数字 {n}");

    // 需要区分错误类型时，再引入 thiserror 或自定义枚举（进阶，见 §9）
    Ok(())
}
```

> `?` 是"语法糖版 match"：`Err(e) => return Err(e.into())`，`Ok(v) => v`。整个文件一路 `?`，出错信息就会带 `Caused by:` 链式打印，足够定位问题。

### 4.6 借用规则：只需记住三条 + `move` 关键字

1. **同一时刻，一个值要么有多个共享引用（`&`），要么有一个独占引用（`&mut`）。**
2. **引用不能活得比它指向的值久。**
3. **默认 move**：把值传给函数/放进闭包 = 所有权转移，原变量失效；需要继续用就 `.clone()`。

```rust
fn main() {
    let s = String::from("hello");

    // 场景：闭包里要拿走一个值（GUI 里极常见）
    let handle = std::thread::spawn(move || {   // move：把 s 的所有权移进闭包
        println!("线程里用 {s}");
    });
    handle.join().unwrap();
    // 此时 s 已经被移走，下面这行会编译报错（如果还需要用，先 let s2 = s.clone();）
    // println!("{s}");

    // 场景：&mut 排他
    let mut x = 5;
    let r1 = &x;          // 共享借用，OK
    // let r2 = &mut x;   // 错误！r1 还活着，不能同时独占借用
    println!("{r1}");
}
```

**生命周期省略规则**：写函数签名时，只要输入只有一个引用、或参数里有 `&self`，输出引用就能省略标注，编译器自动补。所以普通代码里你几乎看不到 `<'_>`：

```rust
fn first_word(s: &str) -> &str {   // 编译器自动理解：返回的引用跟 s 同寿命
    s.split_whitespace().next().unwrap_or("")
}
```

---

## 5. 写脚本：把 Rust 当 Python 用

### 5.1 命令行参数：`std::env`（最简单）

```rust
fn main() {
    let args: Vec<String> = std::env::args().collect();
    // args[0] 是程序路径，args[1..] 是参数
    if args.len() < 2 {
        eprintln!("用法：{} <文件名>", args[0]);   // eprintln 输出到 stderr
        std::process::exit(1);
    }
    let file = &args[1];
    println!("处理 {file}");
}
```

### 5.2 命令行参数：`clap`（专业工具级）

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

```rust
use clap::Parser;

/// 一个简单的文件搜索工具（这段注释就是 --help 的帮助文本）
#[derive(Parser)]
#[command(name = "search", version, about)]
struct Args {
    /// 要搜索的关键字
    keyword: String,

    /// 起始目录，默认当前目录
    #[arg(short, long, default_value = ".")]
    path: String,

    /// 最多输出多少条
    #[arg(short, long, default_value_t = 10)]
    limit: usize,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args = Args::parse();   // 自动解析，参数不合法会自动打印用法并退出
    println!("搜索 {keyword:?}，目录 {}，上限 {}", args.path, args.limit);
    Ok(())
}
```

`--help` 自动生成、参数校验自动完成，这一个小库省掉你手写所有解析代码。

### 5.3 文件与目录操作

```rust
use std::path::PathBuf;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 读写文件
    std::fs::write("out.txt", "内容")?;
    let text = std::fs::read_to_string("out.txt")?;

    // 遍历目录
    let mut names: Vec<String> = std::fs::read_dir(".")?
        .flatten()                       // 跳过读不到的条目
        .filter(|e| e.path().extension().is_some_and(|e| e == "rs"))  // 只要 .rs 文件
        .map(|e| e.file_name().to_string_lossy().to_string())
        .collect();
    names.sort();

    // 路径操作（用 PathBuf，别拼字符串）
    let dir = PathBuf::from("/tmp/project");
    let file = dir.join("src").join("main.rs");   // /tmp/project/src/main.rs
    let parent = file.parent().unwrap();          // /tmp/project/src
    let stem = file.file_stem().unwrap();         // main
    println!("{names:?}");

    // 递归遍历 → 用 walkdir crate（§9）
    Ok(())
}
```

### 5.4 JSON：`serde` + `serde_json`

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Config {
    name: String,
    retries: u32,
    #[serde(default)]          // 字段缺失时用默认值，兼容旧配置
    tags: Vec<String>,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 结构体 → JSON 字符串
    let cfg = Config { name: "app".into(), retries: 3, tags: vec!["cli".into()] };
    let json = serde_json::to_string_pretty(&cfg)?;
    println!("{json}");

    // JSON 字符串 → 结构体（读取配置文件的标准姿势）
    let back: Config = serde_json::from_str(&json)?;
    println!("{back:?}");

    // 懒人模式：不定义结构体，直接当动态对象用
    let v: serde_json::Value = serde_json::from_str(r#"{"ok": true, "items": [1, 2]}"#)?;
    println!("items[0] = {}", v["items"][0]);
    Ok(())
}
```

### 5.5 正则、日期时间

```toml
[dependencies]
regex = "1"
chrono = "0.4"
```

```rust
use regex::Regex;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 正则：匹配 + 提取
    let re = Regex::new(r#"(\d{4})-(\d{2})-(\d{2})"#)?;
    let text = "会议日期 2026-08-04，下次 2026-09-01";
    for caps in re.captures_iter(text) {
        println!("年 {} 月 {} 日 {}", &caps[1], &caps[2], &caps[3]);
    }

    // 日期时间
    let now = chrono::Local::now();
    println!("{}", now.format("%Y-%m-%d %H:%M:%S"));
    let parsed = chrono::NaiveDate::parse_from_str("2026-08-04", "%Y-%m-%d")?;
    println!("{parsed}");
    Ok(())
}
```

### 5.6 调用外部进程

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 运行命令并拿到输出
    let out = std::process::Command::new("git")
        .args(["log", "--oneline", "-5"])
        .output()?;
    if out.status.success() {
        println!("{}", String::from_utf8_lossy(&out.stdout));
    } else {
        eprintln!("{}", String::from_utf8_lossy(&out.stderr));
    }

    // 管道输入（echo 文本进 stdin 等）
    let mut child = std::process::Command::new("sort")
        .stdin(std::process::Stdio::piped())
        .spawn()?;
    use std::io::Write;
    child.stdin.take().unwrap().write_all(b"banana\napple\n")?;
    Ok(())
}
```

### 5.7 HTTP 请求：`reqwest`

```toml
[dependencies]
# 简单脚本用 blocking 版（同步，不用 async）
reqwest = { version = "0.12", features = ["blocking", "json"] }
```

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    // GET
    let resp = reqwest::blocking::get("https://api.github.com/repos/emilk/egui")?
        .json::<serde_json::Value>()?;   // 直接解析成 JSON
    println!("stars: {}", resp["stargazers_count"]);

    // POST（带 JSON body + header）
    let client = reqwest::blocking::Client::new();
    let body = serde_json::json!({ "name": "xxx" });
    let resp = client
        .post("https://httpbin.org/post")
        .header("User-Agent", "my-script/0.1")
        .json(&body)
        .send()?
        .text()?;
    println!("{resp}");
    Ok(())
}
```

想要异步版本？把 `reqwest` 的 `blocking` feature 去掉，加 `tokio`：

```toml
[dependencies]
reqwest = "0.12"
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
```

```rust
#[tokio::main]                      // 属性宏：把一个 async fn 变成真正运行的入口
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let resp = reqwest::get("https://httpbin.org/ip").await?.text().await?;
    println!("{resp}");
    Ok(())
}
```

> 应用开发只需要记住：`#[tokio::main]` + `async fn` + `.await`，三件套。背后的 `Pin`/状态机原理不用管。

### 5.8 综合 CLI 脚本示例：批量重命名文件

把上面的知识串起来，写一个可直接运行的完整脚本：

```rust
// 用法：cargo run -- . "old" "new"    （在目录里把所有含 old 的文件名替换成 new）
use std::path::PathBuf;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut args = std::env::args().skip(1);
    let dir = args.next().unwrap_or_else(|| ".".to_string());
    let find = args.next().unwrap_or_else(|| "old".to_string());
    let replace = args.next().unwrap_or_else(|| "new".to_string());

    let mut renamed = 0;
    let mut failed = 0;

    for entry in std::fs::read_dir(&dir)? {
        let entry = entry?;
        let old_name = entry.file_name().to_string_lossy().to_string();
        if old_name.contains(&find) {
            let new_name = old_name.replace(&find, &replace);
            let from: PathBuf = entry.path();
            let to = from.with_file_name(&new_name);

            if to.exists() {
                println!("跳过（已存在）: {old_name}");
                failed += 1;
                continue;
            }
            match std::fs::rename(&from, &to) {
                Ok(_) => { println!("{old_name} → {new_name}"); renamed += 1; }
                Err(e) => { println!("失败 {old_name}: {e}"); failed += 1; }
            }
        }
    }
    println!("完成：重命名 {renamed} 个，失败 {failed} 个");
    Ok(())
}
```

---

## 6. 开发 GUI：以 egui/eframe 为例

### 6.1 为什么选 egui

（结论来自 2026-08 的生态数据对比，详见上次调研）

| 指标 | egui | 其他纯 Rust 框架 |
|---|---|---|
| crates.io 累计下载 | **约 3700 万**（egui+eframe） | iced 244 万、Slint 142 万、Dioxus 216 万 |
| 近 90 天下载 | **约 840 万** | 其余均在 100 万以内 |
| 维护主体 | Rerun 公司全职维护，自身用于生产级可视化产品 | 多为个人/小团队 |
| 插件生态 | 最丰富（表格、绘图、树、图表……） | 较少 |

egui 是**立即模式（Immediate Mode）**：每帧重新绘制 UI，没有"组件对象树"要管理，状态就是你自己的 Rust 变量。**对于工具类应用，它比保留模式框架写起来快得多，这是它最大的卖点。**

### 6.2 最小窗口程序

```toml
[dependencies]
eframe = "0.31"          # egui 的窗口/运行框架；写桌面应用直接依赖它
```

```rust
use eframe::egui;

fn main() -> eframe::Result {
    let options = eframe::NativeOptions {
        viewport: egui::ViewportBuilder::default()
            .with_inner_size([800.0, 600.0])
            .with_title("我的第一个应用"),
        ..Default::default()
    };
    eframe::run_native(
        "my_app",
        options,
        Box::new(|_cc| Ok(Box::new(MyApp::default()))),   // 返回 Result 的闭包
    )
}

struct MyApp {
    count: i32,
}

impl Default for MyApp {
    fn default() -> Self { Self { count: 0 } }
}

impl eframe::App for MyApp {
    // 每帧调用一次：在这里画 UI、处理事件
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.heading("计数器");
            ui.label(format!("当前值：{}", self.count));
            if ui.button("+1").clicked() {
                self.count += 1;
            }
            if ui.button("清零").clicked() {
                self.count = 0;
            }
        });
    }
}
```

`cargo run` 即可看到窗口。**注意 `update` 里闭包借用 `self` 的问题**：`egui::CentralPanel::show` 的闭包是 `FnMut`，可以捕获 `&mut self`，直接读写字段没问题；但别在闭包里**整体 move** self（比如把 `self` 整个传给别的函数），那是第 10 节要讲的经典报错。

### 6.3 常用控件速览

```rust
use eframe::egui;

struct FormApp {
    name: String,
    password: String,
    memo: String,
    enabled: bool,
    level: f32,
    choice: String,
    choices: Vec<String>,
}

impl Default for FormApp {
    fn default() -> Self {
        Self {
            name: String::new(),
            password: String::new(),
            memo: String::new(),
            enabled: true,
            level: 50.0,
            choice: "A".to_string(),
            choices: vec!["A".to_string(), "B".to_string(), "C".to_string()],
        }
    }
}

impl eframe::App for FormApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            // —— 文本输入 ——
            ui.horizontal(|ui| {
                ui.label("姓名：");
                ui.text_edit_singleline(&mut self.name);        // 单行
            });
            ui.horizontal(|ui| {
                ui.label("密码：");
                ui.add(egui::TextEdit::singleline(&mut self.password).password(true));
            });
            ui.label("备注：");
            ui.add(egui::TextEdit::multiline(&mut self.memo).desired_rows(3));  // 多行

            // —— 开关 / 滑块 / 数字 ——
            ui.checkbox(&mut self.enabled, "启用功能");
            ui.add(egui::Slider::new(&mut self.level, 0.0..=100.0).text("进度"));
            ui.add(egui::DragValue::new(&mut self.level).suffix("%"));

            // —— 下拉选择 ——
            egui::ComboBox::from_label("选项")
                .selected_text(&self.choice)
                .show_ui(ui, |ui| {
                    for c in &self.choices {
                        ui.selectable_value(&mut self.choice, c.clone(), c);
                    }
                });

            // —— 分组面板 ——
            egui::CollapsingHeader::new("高级设置").show(ui, |ui| {
                ui.label("这里的内容默认折叠");
            });

            // —— 提交 ——
            ui.separator();
            if ui.button("提交").clicked() {
                println!("提交：{} {} {}", self.name, self.level, self.choice);
            }
        });
    }
}
```

### 6.4 布局：面板与滚动区域

```rust
impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        // 顶部工具栏（可拖动分隔）
        egui::TopBottomPanel::top("toolbar").show(ctx, |ui| {
            ui.horizontal(|ui| {
                ui.heading("工具");
                if ui.button("刷新").clicked() { /* ... */ }
            });
        });

        // 左侧面板
        egui::SidePanel::left("nav").default_width(200.0).show(ctx, |ui| {
            ui.label("导航");
            if ui.button("页面一").clicked() { /* ... */ }
        });

        // 中央区域
        egui::CentralPanel::default().show(ctx, |ui| {
            egui::ScrollArea::vertical().show(ui, |ui| {
                for i in 0..100 {
                    ui.label(format!("第 {i} 行"));
                }
            });
        });
    }
}
```

### 6.5 表格：`egui_extras`

表格（可排序、列宽可拖）要加 `egui_extras`：

```toml
[dependencies]
egui_extras = "0.31"   # 版本号与 egui 保持一致
```

```rust
use eframe::egui;
use egui_extras::{TableBuilder, Column};

struct Row { name: String, size: u64 }

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        let rows = vec![
            Row { name: "a.txt".into(), size: 120 },
            Row { name: "b.rs".into(), size: 2048 },
        ];
        egui::CentralPanel::default().show(ctx, |ui| {
            TableBuilder::new(ui)
                .column(Column::auto())
                .column(Column::remainder())
                .header(20.0, |mut header| {
                    header.col(|ui| { ui.strong("文件名"); });
                    header.col(|ui| { ui.strong("大小"); });
                })
                .body(|mut body| {
                    for row in &rows {
                        body.row(18.0, |mut row| {
                            row.col(|ui| { ui.label(&row.name); });
                            row.col(|ui| { ui.label(format!("{} B", row.size)); });
                        });
                    }
                });
        });
    }
}
```

### 6.6 绘图：`egui_plot`

```toml
[dependencies]
egui_plot = "0.31"
```

```rust
use eframe::egui;
use egui_plot::{Line, Plot, PlotPoints};

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            Plot::new("sin_plot")
                .height(300.0)
                .show(ui, |plot_ui| {
                    let points: PlotPoints = (0..200)
                        .map(|i| {
                            let x = i as f64 * 0.1;
                            [x, x.sin()]
                        })
                        .collect();
                    plot_ui.line(Line::new(points).name("sin(x)"));
                });
        });
    }
}
```

---

## 7. GUI 进阶：线程通信与文件对话框

### 7.1 后台任务 → UI 更新：`mpsc` 通道（官方推荐模式）

UI 线程不能阻塞。耗时的活（下载、批量处理、大文件分析）放到 `std::thread`，通过**通道**把进度发回 UI。**不要把 `Mutex<Vec>` 到处传，通道更干净。**

```rust
use eframe::egui;
use std::sync::mpsc::{self, Receiver};

enum Msg {
    Progress(f32),
    Finished(String),
}

struct DownloadApp {
    progress: f32,
    result: Option<String>,
    rx: Option<Receiver<Msg>>,   // 存放接收端，避免被 drop
}

impl Default for DownloadApp {
    fn default() -> Self { Self { progress: 0.0, result: None, rx: None } }
}

impl eframe::App for DownloadApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        // 1) 每帧先收取线程发来的消息（非阻塞）
        if let Some(rx) = &self.rx {
            while let Ok(msg) = rx.try_recv() {
                match msg {
                    Msg::Progress(p) => self.progress = p,
                    Msg::Finished(s) => { self.result = Some(s); self.progress = 1.0; }
                }
            }
        }

        egui::CentralPanel::default().show(ctx, |ui| {
            if self.rx.is_none() {
                // 2) 启动后台线程（move 把发送端 tx 移进线程）
                if ui.button("开始下载").clicked() {
                    let (tx, rx) = mpsc::channel();
                    std::thread::spawn(move || {
                        for i in 0..=100 {
                            std::thread::sleep(std::time::Duration::from_millis(50)); // 模拟耗时
                            if tx.send(Msg::Progress(i as f32 / 100.0)).is_err() {
                                break;  // 接收端被 drop（窗口关了），退出
                            }
                        }
                        let _ = tx.send(Msg::Finished("下载完成".to_string()));
                    });
                    self.rx = Some(rx);
                }
            } else {
                // 3) 显示进度
                ui.add(egui::ProgressBar::new(self.progress).show_percentage());
                if let Some(msg) = &self.result {
                    ui.label(msg);
                }
            }
        });
    }
}
```

> 关键点：`try_recv` 非阻塞，不会卡 UI；线程里 `send` 失败就 `break`，避免窗口关闭后线程还在空转。

### 7.2 文件对话框：`rfd`

```toml
[dependencies]
rfd = "0.15"
```

```rust
// 在 update 里的任意 ui 上下文：
if ui.button("选择文件夹").clicked() {
    if let Some(path) = rfd::FileDialog::new().pick_folder() {
        // path: std::path::PathBuf
    }
}
if ui.button("打开文件").clicked() {
    if let Some(path) = rfd::FileDialog::new()
        .add_filter("图片", &["png", "jpg", "jpeg"])
        .set_directory(".")
        .pick_file()
    {
        // ...
    }
}
if ui.button("保存文件").clicked() {
    if let Some(path) = rfd::FileDialog::new().save_file() { /* ... */ }
}
```

### 7.3 打包发布

```sh
cargo build --release
# 产物：target/release/my_app（Linux/macOS 直接可执行；Windows 是 my_app.exe）

# 体积优化：启用 LTO + strip 符号
# Cargo.toml 里加：
# [profile.release]
# lto = true
# strip = true
# codegen-units = 1
```

egui 空窗口 release 体积约 5–15 MB（视平台和优化配置），对工具类软件很友好。要出安装包可以再看 `cargo-bundle` / `tauri-bundler` 这类工具，但日常分发一个可执行文件就够了。

---

## 8. 其他 GUI 框架：何时选它们

| 框架 | 一句话定位 | 渲染方式 | 适合场景 |
|---|---|---|---|
| **egui/eframe** | 立即模式、迭代最快 | 纯 Rust（wgpu/glow/软渲染） | 工具类、内部软件、可视化、游戏编辑器 |
| **Dioxus** | React 风格组件化 | 桌面默认 WebView（原生渲染 Blitz 开发中） | 想要组件树 + 一套代码跨 Web/桌面/移动 |
| **iced** | Elm 架构（消息循环） | 纯 Rust（wgpu） | 喜欢"状态+消息"模式；能接受 0.x API 变动 |
| **Slint** | 声明式 UI + 设计器 | 纯 Rust（femtovg/skia/软渲染） | 嵌入式/低功耗设备；注意 GPL+商业双授权 |
| **Tauri**（非纯 Rust） | Web 技术栈做壳 | 系统 WebView | UI 最漂亮、社区最大；但界面不是 Rust 写的 |

### Dioxus 最小示例（感受下 React 风格）

```toml
[dependencies]
dioxus = "0.6"
```

```rust
use dioxus::prelude::*;

fn App() -> Element {
    let mut count = use_signal(|| 0);   // 响应式状态
    rsx! {
        button { onclick: move |_| count += 1, "点击次数: {count}" }
    }
}

fn main() {
    dioxus::launch(App);
}
```

> 如果你的第一反应是"我要组件复用、跨平台"，看 Dioxus；如果你要"最快把工具写出来"，选 egui。两个都试试，手感差别很大，试完再定。

---

## 9. 常用 crate 速查表

| 用途 | crate | 一句话说明 |
|---|---|---|
| GUI | `eframe` + `egui` + `egui_extras` + `egui_plot` | 桌面 GUI 全家桶（本文件主角） |
| GUI | `rfd` | 原生文件/文件夹选择对话框 |
| CLI 参数 | `clap` | 工业级命令行解析（derive 模式） |
| 终端美化输出 | `colored` / `indicatif` | 颜色 / 进度条 |
| JSON | `serde` + `serde_json` | 序列化事实标准 |
| 配置文件 | `toml` | TOML 解析（`cargo` 同款格式） |
| 日期时间 | `chrono` | 目前最常用；新项目也可看 `time` |
| 正则 | `regex` | 官方维护 |
| HTTP | `reqwest` | 最主流的 HTTP 客户端 |
| 递归目录遍历 | `walkdir` | `read_dir` 只查一层，递归用它 |
| 日志 | `env_logger` + `log` | 调试比 println 专业 |
| 错误类型 | `thiserror` | 为库/应用定义自己的错误枚举 |
| 随机数 | `rand` | 事实标准 |
| 多线程任务调度 | `tokio` | 异步运行时（默认选择） |
| 线程池/并行迭代 | `rayon` | `.par_iter()` 一行并行化 |
| 进程管理 | `duct` | 比 `std::process::Command` 好用 |
| 全局热键/剪贴板/托盘 | `tray-icon` / `arboard` / `global-hotkey` | 桌面小工具三件套 |

---

## 10. 高频编译错误与解法速查

> 借用检查器报错不是打击，是 lint。下面是应用开发中最常碰到的 8 个，每个都有固定解法。

| 报错（节选） | 场景 | 解法 |
|---|---|---|
| `E0382: borrow of moved value` | 把值传进闭包/函数后又用了 | 传之前 `.clone()`；或闭包前加 `move` 并确认不再需要原值 |
| `E0502: cannot borrow ... as immutable because it is also borrowed as mutable` | 同时读和写一个变量 | 先把要读的值拷出来（`let x = self.field.clone();`），或重排代码让两个借用不重叠 |
| `closure requires unique access ... already borrowed` | UI 闭包里又想改 `self` 又想读 `self` 的其他字段 | 把要读的字段先 `let name = self.name.clone();` 拷出来再用（UI 代码最常见） |
| `E0505: cannot move out of ... because it is borrowed` | 循环里借用元素后又移动整个容器 | 遍历 `.clone()` 或用索引循环 |
| `the trait bound \`T: Send\` is not satisfied` / `cannot be sent between threads safely` | 把非 `Send` 的值（如 `Rc`、裸引用）送进线程 | 换成 `Arc`；需要修改用 `Arc<Mutex<T>>`；克隆 `Arc` 后再 `move` |
| `E0597: borrowed value does not live long enough` | 函数返回了指向局部变量的引用 | 别返回引用，返回 owned 类型（`String`/`Vec`）；或用 `'static` 的 `&'static str` |
| `expected \`String\`, found \`&str\`` | 类型不匹配 | `&str` → `String` 用 `.to_string()` / `.into()`；`String` → `&str` 用 `&s` |
| `temporary value dropped while borrowed` | 对临时值取引用存下来 | 先绑定变量：`let tmp = ...; let r = &tmp;` |

**万能起手式**：看到借用报错 → 先在报错行前后找"谁在借用、谁要 move"，多数情况 `.clone()` 一步解决（代价是少量性能，先跑通再优化）。闭包报错 → 检查是否需要 `move`，以及闭包外是否还要用被捕获的值。

---

## 11. 完整综合示例：带 GUI 的批量重命名工具

把第 5 章的 CLI 重命名逻辑和第 6、7 章的 GUI 知识拼起来，就是一个完整可运行的应用：选文件夹 → 预览替换效果 → 一键执行。

```toml
[package]
name = "rename-tool"
version = "0.1.0"
edition = "2021"

[dependencies]
eframe = "0.31"
rfd = "0.15"
```

```rust
use eframe::egui;
use std::path::PathBuf;

#[derive(Default)]
struct RenameApp {
    folder: Option<PathBuf>,
    files: Vec<String>,      // 当前文件夹下的文件名
    find: String,            // 要替换的旧字符串
    replace: String,         // 新字符串
    status: Option<String>,  // 底部提示信息
}

impl RenameApp {
    fn load_folder(&mut self, path: PathBuf) {
        self.files.clear();
        if let Ok(entries) = std::fs::read_dir(&path) {
            for entry in entries.flatten() {
                self.files.push(entry.file_name().to_string_lossy().to_string());
            }
        }
        self.files.sort();
        self.status = Some(format!("已加载 {} 个文件", self.files.len()));
        self.folder = Some(path);
    }

    /// 计算替换后的新文件名；无匹配返回 None
    fn new_name(&self, old: &str) -> Option<String> {
        if self.find.is_empty() {
            return None;
        }
        let new = old.replace(&self.find, &self.replace);
        (new != old).then_some(new)
    }

    fn do_rename(&mut self) {
        let Some(folder) = &self.folder else {
            self.status = Some("请先选择文件夹".to_string());
            return;
        };
        let mut renamed = 0;
        let mut failed = 0;
        for old in &self.files {
            if let Some(new_name) = self.new_name(old) {
                let from = folder.join(old);
                let to = folder.join(&new_name);
                if to.exists() {
                    failed += 1;   // 目标已存在，跳过避免覆盖
                    continue;
                }
                match std::fs::rename(&from, &to) {
                    Ok(_) => renamed += 1,
                    Err(_) => failed += 1,
                }
            }
        }
        self.load_folder(folder.clone());   // 重命名后刷新列表
        self.status = Some(format!("完成：成功 {renamed} 个，失败 {failed} 个"));
    }
}

impl eframe::App for RenameApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        // —— 顶部工具栏：选择文件夹 ——
        egui::TopBottomPanel::top("toolbar").show(ctx, |ui| {
            ui.horizontal(|ui| {
                if ui.button("选择文件夹").clicked() {
                    if let Some(path) = rfd::FileDialog::new().pick_folder() {
                        self.load_folder(path);
                    }
                }
                if let Some(folder) = &self.folder {
                    ui.label(folder.to_string_lossy().to_string());
                }
            });
        });

        egui::CentralPanel::default().show(ctx, |ui| {
            // —— 替换规则 ——
            ui.horizontal(|ui| {
                ui.label("查找:");
                ui.text_edit_singleline(&mut self.find);
                ui.label("替换为:");
                ui.text_edit_singleline(&mut self.replace);
            });
            ui.separator();

            // —— 预览列表（只读借用 self.files，同时调用 &self 方法，可共存）——
            ui.label(format!("文件列表（{}）", self.files.len()));
            egui::ScrollArea::vertical().max_height(300.0).show(ui, |ui| {
                for old in &self.files {
                    ui.horizontal(|ui| {
                        ui.monospace(old);
                        if let Some(new) = self.new_name(old) {
                            ui.label(
                                egui::RichText::new(format!("→ {new}"))
                                    .color(egui::Color32::GREEN),
                            );
                        }
                    });
                }
            });
            ui.separator();

            if ui.button("执行重命名").clicked() {
                self.do_rename();
            }
            if let Some(status) = &self.status {
                ui.label(status);
            }
        });
    }
}

fn main() -> eframe::Result {
    let options = eframe::NativeOptions {
        viewport: egui::ViewportBuilder::default()
            .with_inner_size([720.0, 480.0])
            .with_title("批量重命名工具"),
        ..Default::default()
    };
    eframe::run_native(
        "rename-tool",
        options,
        Box::new(|_cc| Ok(Box::new(RenameApp::default()))),
    )
}
```

试试扩展方向（都是几行代码的事）：
- 用 `egui_extras` 表格替换预览列表，支持按列排序；
- 用 `regex` 支持"正则替换"（把 `new_name` 里的 `replace` 换成正则）；
- 用 `mpsc` 把重命名放到后台线程，文件多时 UI 不卡；
- 加一个"撤销"按钮：执行前把 `(from, to)` 对存起来。

---

## 12. 务实学习路线

**总原则：学 20% 的知识，解决 80% 的问题，剩下的遇到再查。**

1. **第 1–2 周**：官方 [The Rust Book](https://doc.rust-lang.org/book/) 前 8 章（所有权、结构体、枚举、错误处理、集合、模块）。跳过宏、unsafe、高级 trait。配合 [rustlings](https://github.com/rust-lang/rustlings) 做练习（每天 15 分钟）。
2. **第 3 周**：抄一遍本文第 5 章的脚本示例，改成你自己的小工具（处理你手头真实的数据/文件）。目标：熟练 `?`、`Result`、迭代器。
3. **第 4 周**：抄 egui 最小程序，然后照着第 11 章的综合示例做你的第一个 GUI。遇到借用报错翻第 10 节速查表。
4. **之后**：按需学 async/tokio、`thiserror`、测试（`cargo test`）。GUI 深入学习直接看 [egui 官方 demo](https://github.com/emilk/egui/tree/master/crates/egui_demo_app) 的源码——它是最好的教程。

**检查你是否可以毕业**：不看资料，30 分钟内用 egui 写一个"输入名字 → 点击保存 → 追加写入 CSV 文件 → 表格显示已有记录"的小工具，并且能通过 `cargo clippy`。做到这步，你已经能独立开发大部分工具类 GUI 和脚本了。

---

*本文是《Rust 深度剖析》的务实补充篇：深度剖析讲"Rust 为什么这样设计"，本文讲"怎么用最少的知识把它用起来"。两篇配合阅读，先实战建立手感，再回头理解原理，效率最高。*
