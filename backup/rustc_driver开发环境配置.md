本文旨在记录如何开发rustc_driver，并在vscode上实现编译器源码代码的提示和跳转

## Rust安装
```bash
# 安装rustup管理rust的工具链
~ curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
# 查看安装是否成功
~ rustup toolchain list
stable-x86_64-unknown-linux-gnu (active, default)
~ rustc -vV
rustc 1.85.0 (4d91de4e4 2025-02-17)
binary: rustc
commit-hash: 4d91de4e48198da2e33413efdcd9cd2cc0c46688
commit-date: 2025-02-17
host: x86_64-unknown-linux-gnu
release: 1.85.0
LLVM version: 19.1.7
# 安装nightly版本, 开发插件只能用nightly版本
~ cargo +nightly --version
```
vscode中需要安装rust-analyzer插件
如果需要借助vscode进行调试，还需要安装codelldb

## custom rustc_driver
```bash
~ cargo new my_rustc_driver
~ cd my_rustc_driver
~ cargo run
# 项目输出hello world
```
复制下面代码到src/main.rs中
```rust
#![feature(rustc_private)]

extern crate rustc_driver;
fn main() {
    println!("this is a custom driver!");
    rustc_driver::main();
}
```
再次运行cargo run会报错，因为rustc_driver找不到。
在项目根路径下创建rust-toolchain.toml
```toml
[toolchain]
# 查看自己的rustc版本
channel = "nightly-2025-02-17"
components = ["cargo", "llvm-tools", "rust-src", "rust-std", "rustc", "rustc-dev", "rustfmt"]
```
再次执行cargo run, 发现已经可以成功运行了
但是vscode的rust-analyzer仍然提示找不到rustc_driver, 而且也无法对rustc内部的crate进行提示和跳转
在项目根路径下创建.vscode/setting.json
```json
{
    "rust-analyzer.rustc.source": "discover",
}
```
在Cargo.toml中添加下面内容
```toml
[package.metadata.rust-analyzer]
rustc_private = true
```
等待cargo check运行完毕，发现已经可以正确跳转了，例如可以跳转到rust_driver::main函数内部查看实现。

如果想在vscode中进行单步调试，需要在.vscode/settings.json中设置一下动态库的链接路径

有了代码跳转，就可以开始愉快地学习rustc编译器内部的实现了！
