<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 简介

欢迎使用**godot-rust**文档！这是一本关于**gdext**（Godot 4的Rust绑定）的用户指南，目前仍在更新中。

如果你是Rust的新手，强烈建议在开始之前先了解一下[Rust的官方文档](https://doc.rust-lang.org/book/)。

你可能感兴趣的其他资源：

📘 [最新的API文档][api-docs]  
⚗️ [Demo项目][demo-projects]  
📄 [英文文档][book-en]  
📔 [gdnative手册 (Godot3绑定)][gdnative-book]  


## godot-rust的目的

Godot 是一个功能齐全的游戏引擎，能够促进高效且有趣的游戏开发工作流程。它内置了`GDScript`作为脚本语言，并且官方也支持`C++`和`C#`绑定。其`GDExtension`机制允许集成更多语言，Rust就是其中之一。

Rust为游戏开发带来了现代化、健壮且高效的体验。如果你对可扩展性、强类型系统感兴趣，或者只是喜欢Rust这门语言，你可以考虑将其与Godot结合使用，享受两者的最佳特性。

另请参阅 [理念][philosophy]以了解更多关于 godot-rust 背后的核心理念。


## 关于本项目

godot-rust是一个由[社区开发][github-contributors]的开源项目。它独立于Godot本身进行维护，但我们与引擎开发人员保持密切联系，促进思想的持续交流。这使我们能够在上游解决许多Rust的需求，同时也在多个方面改善了引擎本身。


### 当前支持的功能

有关实现状态的最新概况，请参考[issue #24][features]。


### 术语

为了避免混淆，以下是你在本书中可能遇到的一些名称和技术的解释：

- [**godot-rust**][ref-godot-rust]: 包含Godot 3和4的Rust绑定以及相关工作的整个项目（书籍、社区等）。
- [**gdext**][github-gdext] (小写): GDExtension（Godot 4）的Rust绑定——本书的重点。
- [**gdnative**][github-gdnative] (小写): ：GDNative（Godot 3）的Rust绑定。
- [**GDExtension**][ref-godot-gdext]: Godot 4提供的C API。
- [**GDNative**][ref-godot-gdnative]:  Godot 3提供的C API。
- **Extension**: 扩展是一个动态的C库，由任何语言绑定（Rust、C++、Swift等）开发。它使用`GDExtension` API，并可以被Godot 4加载。

以下为错误术语：`GDRust`、`gdrust`、`godot-rs`。

[features]: https://github.com/godot-rust/gdextension/issues/24

[api-docs]: https://godot-rust.github.io/docs/gdext
[book-en]: https://godot-rust.github.io/book
[demo-projects]: https://github.com/godot-rust/demo-projects
[gdnative-book]: https://godot-rust.github.io/gdnative-book
[github-contributors]: https://github.com/godot-rust/gdext/graphs/contributors
[github-gdext]: https://github.com/godot-rust/gdext
[github-gdnative]: https://github.com/godot-rust/gdnative
[ref-godot-gdext]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/what_is_gdextension.html
[ref-godot-gdnative]: https://docs.godotengine.org/en/3.5/tutorials/scripting/gdnative/what_is_gdnative.html
[ref-godot-rust]: https://godot-rust.github.io/
[philosophy]: contribute/philosophy
