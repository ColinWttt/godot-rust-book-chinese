<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 注册 Rust 符号

本章节介绍如何将自己的 Rust 代码暴露给 Godot。你通过在引擎中 _注册_ 单独的符号（类、函数等）来实现这一点。

从类注册开始，章节接着详细介绍如何注册函数、属性、信号和常量。


<!-- TODO: Futher aspects cover the Rust-to-Godot conversions using `ToGodot`/`FromGodot` traits and the registration of enums. -->


## 过程宏 API

目前，过程宏 API 是注册 Rust 符号的唯一方式。提供了多种过程宏（派生和属性宏）来修饰你的 Rust 项，如 `struct` 或 `impl` 块。在底层，这些宏会生成必要的胶水代码，将每个项目注册到 Godot 中。

该库的设计方式是，你可以利用现有的所有知识，并通过宏语法扩展它，而不需要学习完全不同的做事方式。我们尽量避免使用外部的 DSL（领域特定语言），而是基于 Rust 的现有语法进行构建。

这种方法有效地减少了你需要编写的样板代码，从而让你更容易专注于重要部分。例如，你很少需要重复自己的工作或在多个地方注册同一个内容（例如，声明一个方法、在另一个 `register` 方法中提到它，并且再次以字符串字面量的形式重复其名称）。


## “导出” ("exporting")

“导出”这个术语有时会被误用。如果你指的是“注册”，请避免谈论“导出类”或“导出方法”。这种说法往往会引起混淆，尤其是对于初学者来说。

在 Godot 的情景中，_导出_ 已经有两个明确的定义：

1. 导出属性。这并不将属性 _注册_ 到 Godot，而是让它在编辑器中可见。
    •    GDScript 使用 `@export` 注解来实现这一点，而我们使用 `#[export]`。
    •    另请参见 [GDScript 导出属性][godot-export-properties]。
2. 导出项目，即将项目打包为发布版本。
    •    编辑器提供了一个 UI，用于构建游戏或应用程序的发布版本，以便它们作为独立的可执行文件运行。
构建可执行文件的过程称为“导出”。
    •    另请参见 [导出项目][godot-export-projects]。

[godot-export-properties]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_exports.html
[godot-export-projects]: https://docs.godotengine.org/en/stable/tutorials/export/index.html
