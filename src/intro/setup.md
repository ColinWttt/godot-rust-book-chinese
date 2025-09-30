<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 设置

在开始编写 Rust 代码之前，我们需要安装一些工具。


## Godot 引擎

虽然可以在没有 Godot 引擎的情况下编写 Rust 代码，但我们强烈推荐安装 Godot，以便快速获得反馈。
在本教程的其余部分，我们假设您已经安装了 Godot 4，并且可以通过以下方式之一访问它：

- 在您的 `PATH` 设置 `godot4`，
- 或者设置一个名为 `GODOT4_BIN` 的环境变量，包含 Godot 可执行文件的路径。


### 从预构建的二进制文件安装 Godot

您可以[从官方网站][godot-download]下载 Godot 4 的二进制文件。
对于 Beta 版本和旧版本，您也可以查看 [下载归档][godot-download-archive]。


### 通过命令行安装 Godot

```bash
# --- Linux ---
# 对于 Fedora/RHEL.
dnf install godot

# Arch Linux.
pacman -Syu godot
paru -Syu godot

# 通过 Flatpak 安装，适用于所有发行版
flatpak install flathub org.godotengine.Godot


# --- Windows ---
# 可以通过 WinGet 安装 Windows 版本
winget install -e --id GodotEngine.GodotEngine
choco install godot
scoop bucket add extras && scoop install godot


# --- macOS ---
brew install godot
```

```admonish note title="其他 Godot 版本"
如果您打算使用不同于最新稳定版本的 Godot 版本，请阅读 [选择 Godot 版本][godot-version]。
```


## Rust

[rustup] 是安装 Rust 工具链的首选方式。它包含了编译器、标准库、Cargo（包管理器），以及如 rustfmt 或 clippy 等工具。请访问官网以下载适合您平台的二进制文件或安装程序。或者，您也可以通过命令行安装。


### 通过命令行安装 rustup

```bash
# Linux (distro-independent)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Windows
winget install -e --id Rustlang.Rustup

# macOS
brew install rustup
```

安装 `rustup` 和 `stable` 工具链后，您可以通过以下命令验证它们是否工作正常：

```bash
$ rustc --version
rustc 1.88.0 (6b00bc388 2025-06-23)
```


## LLVM

```admonish tip
通常来说，您**不**需要安装 LLVM。
```

过去，由于 `bindgen` [依赖于 LLVM][llvm-bindgen]，因此需要安装它。
不过，现在我们提供了预构建的构件，因此大多数用户只需添加 Cargo 依赖项并立即开始使用，这样做显著减少了初始的编译时间，因为 `bindgen`以前依赖了许多传递性的依赖项，导致体积较大。

如果您计划使用 `api-custom` 功能，例如拥有一个分叉的 Godot 版本或自定义模块，则仍然需要 LLVM。
但如果您只打算使用不同版本的 Godot API，则 _无_ 需安装 LLVM；详情请见 选择 [Godot 版本][godot-version]。

LLVM 的二进制文件可以从 [llvm.org][llvm] 下载。安装后，您可以检查 LLVM 的 clang 编译器是否可用：

```bash
clang -v
```


[godot-download-archive]: https://godotengine.org/download/archive/
[godot-download]: https://godotengine.org/download/
[godot-version]: ../toolchain/godot-version.md
[llvm-bindgen]: https://rust-lang.github.io/rust-bindgen/requirements.html
[llvm]: https://releases.llvm.org
[rustup-windows]: https://github.com/rust-lang/rustup#working-with-rust-on-windows
[rustup]: https://rustup.rs
