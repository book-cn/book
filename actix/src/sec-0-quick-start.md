# 快速开始

在开始编写 actix 应用程序之前需要先安装某一版本的 Rust。
建议使用 rustup 来安装或配置版本。

## 安装 Rust

在开始之前，我们需要使用 [rustup](https://rustup.rs/) 安装程序来安装 Rust：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

如果已经安装过 rustup，请运行这个命令来确保拥有最新版本的 Rust：

```bash
rustup update
```

actix 框架需要 Rust 1.40.0 及更高版本。

## 运行示例

开始试验 actix 最快的方法是克隆 actix 版本库<!--
-->并运行 examples/ 目录中所包含的示例。以下这组<!--
-->命令会运行 `ping` 示例：

```bash
git clone https://github.com/actix/actix
cargo run --example ping
```

更多示例请查看 [examples/](https://github.com/actix/actix/tree/master/examples) 目录。
