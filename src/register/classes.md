<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 类的注册

类是 Godot 数据建模的核心。如果你想以类型安全的方式构建复杂的用户定义类型，就无法避免使用类。数组、字典和简单类型只能满足基本需求，过度使用它们会使得使用静态类型语言的意义大打折扣。

Rust 使得类注册变得简单。如前所述，Rust 语法作为基础，并加入了特定于 gdext 的扩展。

另请参见 [GDScript 类参考][godot-gdscript-classes]。


## 目录

<!-- toc -->


## 定义一个 Rust 结构体

在 Rust 中，Godot 类由结构体表示。结构体的定义方式与平常一样，可以包含任意数量的字段。为了将它们注册到 Godot 中，你需要派生 `GodotClass` trait。

```admonish info title="GodotClass trait"
`GodotClass` trait标记了所有在 Godot 中已知的类。例如，`Node` 或 `Resource` 类已经实现了该trait。
如果你想注册自己的类，也需要实现 `GodotClass` trait。

`#[derive(GodotClass)]` 简化了这个过程，并处理了所有的样板代码。  
有关详细信息，请参见 [API 文档][api-derive-godotclass]。

```

让我们定义一个简单的类，命名为  `Monster`：

```rust
#[derive(GodotClass)]
#[class(init)] // 稍后会详细说明这个
struct Monster {
    name: String,
    hitpoints: i32,
}
```

就这样。在编译后，这个类通过热重载（在 Godot 4.2 之前是重启后）会立即在 Godot 中可用。虽然它还不太有用，但上述定义足以将 `Monster` 类注册到引擎中。

```admonish info title="自动注册"
`#[derive(GodotClass)]` 会 _自动_ 注册该类 —— 你无需显式调用 `add_class()` 注册或维护一个包含所有类的中央列表。

该过程宏会在启动时自动将类注册到一个这样的列表中。
```


## 选择基类

默认情况下，Rust 类的基类是 `RefCounted`。这和 GDScript 在省略 `extends`  关键字时的行为一致。

`RefCounted` 对于数据包非常有用。顾名思义，它允许通过引用计数器来共享实例，因此你不需要担心内存管理。`Resource` 是 `RefCounted` 的子类，适用于需要序列化到文件系统的数据。

然而，如果你希望你的类成为场景树的一部分，则需要使用 `Node`（或其派生类）作为基类。

这里，我们使用一个更具体的节点类型 `Node3D`。这可以通过在结构体定义上指定 `#[class(base=Node3D)]` 来实现：


```rust
#[derive(GodotClass)]
#[class(base=Node3D)]
struct Monster {
    name: String,
    hitpoints: i32,
}
```


## 基类字段

由于 Rust 没有继承机制，我们需要使用组合来实现相同的效果。gdext 提供了一个 `Base<T>`类型，它允许我们将 Godot 超类（基类）的实例作为字段存储在我们的 `Monster` 类中。

```rust
#[derive(GodotClass)]
#[class(base=Node3D)]
struct Monster {
    name: String,
    hitpoints: i32,
    base: Base<Node3D>,
}
```

关键部分是 `Base<T>` 类型。`T` 必须与你在 `#[class(base=...)]` 属性中指定的基类相匹配。你也可以使用关联类型 `Self::Base` 来代替 `T`。

当你在结构体中声明基类字段时，`#[derive]` 过程宏会自动检测到 `Base<T>` 类型。[^inference]
这使得你可以通过提供的方法 `self.base()` 和 `self.base_mut()` 访问 `Node` API，稍后我们会详细介绍。


## 结论

你已经学会了如何定义一个 Rust 类并将其注册到 Godot 中。你现在知道了不同的基类存在，并且了解如何选择一个基类。

接下来的章节将介绍函数和构造函数。

<br>

---

[^inference]: 您可以使用 `#[hint]` 属性调整类型检测，详见[相关文档][api-derive-godotclass-inference]。


[api-derive-godotclass]: https://godot-rust.github.io/docs/gdext/master/godot/register/derive.GodotClass.html
[api-derive-godotclass-inference]: https://godot-rust.github.io/docs/gdext/master/godot/register/derive.GodotClass.html#fine-grained-inference-hints
[api-godot-api]: https://godot-rust.github.io/docs/gdext/master/godot/register/attr.godot_api.html
[godot-gdscript-classes]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#classes
