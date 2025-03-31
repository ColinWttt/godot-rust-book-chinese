<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 调用函数

一般来说，gdext 库以尽可能符合 Rust 风格的方式映射 Godot 函数。有时，函数签名（signatures）会与 GDScript 略有不同，本文将详细介绍这些差异。


## 目录
<!-- toc -->

## Godot 类

Godot 类位于 `godot::classes` 模块中。一些常用的类，如 `Node`、`RefCounted`、`Node3D` 等，还会在 `godot::prelude` 中重新导出。

Godot 的大部分功能是通过类内部的函数暴露的。请随时查看 [API 文档][api-classes] 以获取更多信息。


## Godot 函数

像 Rust 中通常一样，函数分为 _方法_（带有 `&self`/`mut self`接收者）和 _关联函数_（在 Godot 中称为“静态函数”）。

要访问 `Gd<T>` 指针上的 Godot API，只需直接在 `Gd`对象上调用方法即可。这是因为 `Deref` 和 `DerefMut` trait，它们通过 `Gd` 为你提供对象引用。

在 [后续章节][book-function-objects] 中，我们还将看到如何调用在 Rust 中定义的函数。

```rust
// Call with &self receiver.
let node = Node::new_alloc();
let path = node.get_path();

// Call with &mut self receiver.
let mut node = Node::new_alloc();
let other: Gd<Node> = ...;
node.add_child(other);
```

方法是否需要共享引用（`&T`）或可变引用（`&mut T`）取决于该方法在GDExtension API中的声明方式（是否为`const`）。请注意，这种区分仅为**信息性**的，不涉及安全性问题，但在实践中，它有助于检测意外的修改。技术上，你总是可以通过`Gd::clone()`创建另一个指针。

关联函数（在GDScript中称为“静态函数”）是直接在类型本身上调用的。


```rust
Node::print_orphan_nodes();
```


## 单例（Singletons）

单例类（不要与有时也被称为单例的 _自动加载_ 混淆）提供了一个`singleton()`函数来访问唯一的实例。方法在该实例上调用：

```rust
let input = Input::singleton();
let jump = input.is_action_pressed("jump");
let mouse_pos = input.get_mouse_position();

// 可变操作需要使用mut:
let mut input = input;
input.set_mouse_mode(MouseMode::CAPTURED);
```

关于是否直接在单例类型上提供方法，而不需要调用`singleton()`，有[讨论][issue-singleton-no-receiver]。然而，这样做会丢失可变性信息，并且还有一些其他问题。


## 默认参数

GDScript支持参数的默认值。如果没有传递参数，则使用默认值。作为例子，我们可以使用[`AcceptDialog.add_button()`][godot-acceptdialog-add-button]。GDScript签名如下：

```GDScript
Button add_button(String text, bool right=false, String action="")
```

因此，你可以在GDScript中以以下方式调用：

```GDScript
var dialog = AcceptDialog.new()
var button0 = dialog.add_button("Yes")
var button1 = dialog.add_button("Yes", true)
var button2 = dialog.add_button("Yes", true, "confirm")
```

在Rust中，我们仍然有一个基本方法[`AcceptDialog::add_button()`][api-acceptdialog-add-button]，它没有默认参数。
它可以像平常一样调用：


```rust
let dialog = AcceptDialog::new_alloc();
let button = dialog.add_button("Yes");
```

因为Rust不支持默认参数，我们必须通过不同的方式模拟其他调用。我们决定使用构建者模式。

在gdext中，构建器方法以 **`_ex后缀`** 命名。这样的一个方法接收所有必需的参数，就像基本方法一样。它返回一个构建器对象，提供按名称设置可选参数的方法。最终，`done()`方法结束构建并返回Godot函数调用的结果。

例如，方法[`AcceptDialog::add_button_ex()`][api-acceptdialog-add-button-ex]。以下这两种调用完全等效：

```rust
let button = dialog.add_button("Yes");
let button = dialog.add_button_ex("Yes").done();
```

你还可以通过构建器对象的方法传递可选参数。只需指定所需的参数。
这里的优点是，你可以使用任意顺序，并且跳过任何参数——不同于GDScript，在GDScript中只能跳过最后的参数。

```rust
// GDScript: dialog.add_button("Yes", true, "")
let button = dialog.add_button_ex("Yes")
    .right(true)
    .done();

// GDScript: dialog.add_button("Yes", false, "confirm")
let button = dialog.add_button_ex("Yes")
    .action("confirm")
    .done();

// GDScript: dialog.add_button("Yes", true, "confirm")
let button = dialog.add_button_ex("Yes")
    .right(true)
    .action("confirm")
    .done();
```


## 动态调用

有时你可能想调用那些在Rust API中没有暴露的函数。这些可能是你在自定义GDScript代码中编写的函数，或来自其他GDExtensions的方法。

当你没有静态信息时，你可以使用Godot的反射API。Godot提供了[`Object.call()`][godot-object-call]等方法，在Rust中通过两种方式暴露。

如果你期望调用成功（因为你知道你编写的GDScript代码），可以使用[`Object::call()`][api-object-call]。如果调用失败，这个方法会panic并提供详细的消息。


```rust
let node = get_node_as::<Node2D>("path/to/MyScript");

// Declare arguments as a slice of variants.
let args = &["string".to_variant(), 42.to_variant()];

// Call the method dynamically.
let val: Variant = node.call("my_method", args);

// Convert to a known type (may panic; try_to() doensn't).
let vec2 = val.to::<Vector2>();
```

如果你想处理调用失败的情况，可以使用 [`Object::try_call()`][api-object-trycall]。这个方法返回一个 `Result` ，其中包含结果或`CallError`错误。

```rust
let result: Result<Variant, CallError> = node.try_call("my_method", args);

match result {
    Ok(val) => {
        let vec2 = val.to::<Vector2>();
        // ...
    }
    Err(err) => {
        godot_print!("Error calling method: {}", err);
    }
}
```

[api-acceptdialog-add-button-ex]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.AcceptDialog.html#method.add_button_ex
[api-acceptdialog-add-button]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.AcceptDialog.html#method.add_button
[api-classes]: https://godot-rust.github.io/docs/gdext/master/godot/classes/index.html
[api-object-call]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Object.html#method.call
[api-object-trycall]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Object.html#method.try_call
[book-function-objects]: ../register/functions.html#methods-and-object-access
[godot-acceptdialog-add-button]: https://docs.godotengine.org/en/stable/classes/class_acceptdialog.html#class-acceptdialog-method-add-button
[godot-object-call]: https://docs.godotengine.org/en/stable/classes/class_object.html#class-object-method-call
[issue-singleton-no-receiver]: https://github.com/godot-rust/gdext/issues/127
