<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 简介

欢迎使用**godot-rust**文档！这是一本关于**gdext**（Godot 4的Rust绑定）的用户指南。

如果你是Rust的新手，强烈建议在开始之前先了解一下[Rust的官方文档](https://doc.rust-lang.org/book/)。

你可能感兴趣的其他资源：

📘 [最新的API文档][api-docs]  
⚗️ [Demo项目][demo-projects]  
📄 [英文文档][book-en]  
📔 [gdnative手册 (Godot3绑定)][gdnative-book]  


## godot-rust的目的

Godot是一款包含多种功能的游戏引擎，能够促进高效且有趣的游戏开发工作流程。它内置了`GDScript`作为脚本语言，并且官方也支持`C++`和`C#`绑定。其`GDExtension`机制允许更多语言集成，Rust就是其中之一。

Rust为游戏开发带来了现代化、健壮且高效的体验。如果你对可扩展性、强类型系统感兴趣，或者只是喜欢Rust这门语言，你可以考虑将其与Godot结合使用，享受两者的最佳特性。


## 关于本项目

godot-rust是一个由[社区开发][github-contributors]的开源项目。它独立于Godot本身进行维护，但我们与引擎开发人员保持密切联系，促进思想的持续交流。这使我们能够在上游解决许多Rust的需求，同时也在多个方面改善了引擎本身。


### 当前支持的功能

有关实现状态的最新概况，请参考[issue #24][features]。


### 术语

为了避免混淆，以下是你在本书中可能遇到的一些名称和技术的解释：

- [**godot-rust**][ref-godot-rust]: 包含Godot 3和4的Rust绑定以及相关工作的整个项目（书籍、社区等）。
- [**GDExtension**][ref-godot-gdext]: Godot 4提供的C API。
- [**GDNative**][ref-godot-gdnative]:  Godot 3提供的C API。
- [**gdext**][github-gdext] (小写): GDExtension（Godot 4）的Rust绑定——本书的重点。
- [**gdnative**][github-gdnative] (小写): ：GDNative（Godot 3）的Rust绑定。
- **Extension**: 扩展是一个动态的C库，由任何语言绑定（Rust、C++、Swift等）开发。它使用`GDExtension` API，并可以被Godot 4加载。


### GDExtension API：新特性

本节简要介绍从功能角度看，Godot 3和4的原生接口之间的差异。

尽管底层的FFI（外部函数接口）层已经完全重写，但从用户的角度来看，许多概念保持不变。特别是，Godot采用基于节点的场景图（scene graph）方法，由具有继承关系的类组成，这一点没有变化。

不过，确实有一些显著的差异：


1. **原生脚本 ⇾ 扩展类**

   在GDNative中，Rust类可以作为 _原生脚本_ 注册。这些脚本可以附加到节点上，以增强其功能，类似于GDScript脚本的附加方式。而GDExtension则直接支持将Rust类型作为引擎类，参见下一点。

   在将GDScript代码迁移到Rust时，请记住，使用Rust代码的首选方式是通过类，而不是脚本。得益于出色的贡献，我们目前 _确实_ 支持[Rust脚本][api-obj-script]，尽管它们的开发程度不如类。


2. **头等对象（First-class citizen）类型**

   在Godot 3中，用户定义的原生类在编辑器中有很多限制：类型注解不完全支持，它们不能轻松用作自定义`resources`等。而通过GDExtension，Rust中用户定义的类，行为上更接近GDScript类。它们也不再需要单独的`.gdns`文件来注册。

3. **始终开启（Always-on）**

   GDNative区分了“工具”脚本和“普通”脚本。在GDExtension中，原生逻辑默认在Godot编辑器启动时就运行，但godot-rust明确改变了这种行为。
   在Rust中，所有虚拟回调（`ready`、`process`等）默认在编辑器模式下不会调用。可以通过`#[class(tool)]`和[`ExtensionLibrary`][extension-library-doc]trait来配置此行为。

4. **编辑器打开时无需重新编译**

   在Godot 4.2之前，在编辑器启动后，无法重新编译Rust库并让更改生效。编辑器重新加载功能已实现，详情请见 [issue #66231]。


[features]: https://github.com/godot-rust/gdextension/issues/24
[issue #66231]: https://github.com/godotengine/godot/issues/66231
[extension-library-doc]: https://godot-rust.github.io/docs/gdext/master/godot/init/trait.ExtensionLibrary.html#method.editor_run_behavior

[api-docs]: https://godot-rust.github.io/docs/gdext
[api-obj-script]: https://godot-rust.github.io/docs/gdext/master/godot/obj/script/index.html
[book-en]: https://godot-rust.github.io/book
[demo-projects]: https://github.com/godot-rust/demo-projects
[gdnative-book]: ../gdnative-book
[github-contributors]: https://github.com/godot-rust/gdext/graphs/contributors
[github-gdext]: https://github.com/godot-rust/gdext
[github-gdnative]: https://github.com/godot-rust/gdnative
[ref-godot-gdext]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/what_is_gdextension.html
[ref-godot-gdnative]: https://docs.godotengine.org/en/3.5/tutorials/scripting/gdnative/what_is_gdnative.html
[ref-godot-rust]: https://godot-rust.github.io/
