<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 编辑器插件

使用 `EditorPlugin` 类型非常类似于 [在 GDScript 中编写插件][gd-plugins] 的过程。

与 GDScript 插件不同，godot-rust 插件会自动注册，并且无法在项目设置的插件面板中启用/禁用。

在 GDScript 中编写的插件如果存在代码错误会自动禁用，但由于 Rust 是一种编译语言，您无法引入编译时错误。


[gd-plugins]: https://docs.godotengine.org/en/stable/tutorials/plugins/editor/making_plugins.html


## 创建 `EditorPlugin`

```rust
#[derive(GodotClass)]
#[class(tool, init, base=EditorPlugin)]
struct MyEditorPlugin {
    base: Base<EditorPlugin>,
}

#[godot_api]
impl IEditorPlugin for MyEditorPlugin {
    fn enter_tree(&mut self) {
    // 在这里执行典型的插件操作。
    }

    fn exit_tree(&mut self) {
    // 在这里执行典型的插件操作。
    }
}
```

由于这是一个 `EditorPlugin`，它会自动被添加到场景树的根节点。这意味着它可以在运行时访问场景树。此外，可以通过此节点安全地访问 `EditorInterface` 单例，这允许您直接向编辑器添加不同的 GUI 元素。如果您想实现一个复杂的 GUI，这会非常有帮助。

<!-- TODO: more plugins from https://docs.godotengine.org/en/stable/tutorials/plugins/editor/index.html -->