<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 实用案例


## 自定义 resources

使用 godot-rust，您可以定义自定义的 `Resource` 类，这些类随后可以供终端用户使用。


## 编辑器插件

`EditorPlugin` 类型在编辑器和运行时加载，并能够访问编辑器和场景树。该类型的功能与用 GDScript 编写的典型 `EditorPlugin` 类相同，但关键在于它可以访问 _整个 Rust 生态系统_。


## 引擎单例

引擎单例是一个始终全局可用的类实例（遵循单例模式）。然而，它无法通过任何可靠的方式访问 `SceneTree`。


## `ResourceFormatSaver` 和 `ResourceFormatLoader`

提供自定义的逻辑来保存和加载您的 `Resource` 派生类。


## 自定义图标

为您的类添加自定义图标实际上是非常简单的！
