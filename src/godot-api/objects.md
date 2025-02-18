<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 对象

这章介绍了 Rust 绑定中最核心的机制 —— 从 Hello-World 示例到复杂的 Rust 游戏开发过程中，都会伴随你的一项机制。

我们说的是 _对象_ 以及它们如何与 Godot 引擎进行集成。


## 目录
<!-- toc -->


## 术语

为了避免混淆，每当我们提到对象时，我们指的是 _Godot类的实例_。这包括 `Object`（层级结构的根类）以及所有直接或间接继承自它的类：`Node`、`Resource`、`RefCounted` 等。

特别地，“类” 这个术语也包括那些通过 `#[derive(GodotClass)]` 声明的用户自定义类型，即使 Rust 从技术上讲称它们为结构体。同样，_继承_ 指的是概念上的关系（如 “`Player` 继承自 `Sprite2D`”），而不是任何技术语言实现上的继承。

对象**不**包括像 `Vector2`, `Color`, `Transform3D`, `Array`, `Dictionary`等内置类型。尽管这些类型有时被称为“内置类”，它们并不是真正的类，我们通常不会将它们的实例称作 _对象_。


### 继承

继承是 Godot 中的一个核心概念。你可能已经通过节点层级结构了解过它，派生类在其中添加了特定的功能。这个概念同样扩展到了 Rust 类，在 Rust 中，继承通过组合来模拟。

每个 Rust 类都有一个 Godot 基类。

- 通常，基类是节点类型，即它（间接地）继承自 `Node` 类。这使得可以将该类的实例附加到场景树中。节点是手动管理的，因此你需要将它们添加到场景树中或手动释放它们。
- 如果没有明确指定，基类是 `RefCounted`。这对于在不与场景树交互的情况下移动数据非常有用。一般来说，“数据集合”（多个字段组成，但没有太多逻辑）应使用 RefCounted。
- `Object` 是继承树的根类。它很少被直接使用，但它是 `Node` 和 `RefCounted` 的基类。仅当你真的需要时才使用它，因为它需要手动内存管理，而且更难处理。


```admonish note title="继承自定义基类"
你不能继承其他 Rust 类或 GDScript 中声明的用户定义类。

要在 Rust 类之间创建关系，请使用组合和特征。这个库在这方面仍在[一些探索中][issue-traits]，
因此抽象 Rust 类的最佳实践可能在未来会有所变化。

```


## `Gd` 智能指针

[`Gd<T>`][api-gd] 是你在使用 gdext 时最常遇到的类型。

它也是库提供的最强大和最灵活的类型。

具体来说，它的职责包括：

- 持有对 _所有_ Godot 对象的引用，无论它们是像 `Node2D` 这样的引擎类型，还是你自己在 Rust 中定义的 `#[derive(GodotClass)]` 结构体。
- 追踪引用计数(reference-counted)类型的内存管理。
- 通过内部可变性安全地访问用户定义的 Rust 对象。
- 检测销毁的对象并防止未定义行为（如双重释放、悬空指针等）。
- 提供 Rust 和引擎代表之间的 FFI 转换，适用于引擎提供和用户暴露的 API。


以下是一些实际示例（即使你还没有完全理解它们，也不要担心，稍后会详细解释）：

1. 获取当前节点的子节点——类型推导为 `Gd<Node3D>`：
    ```rust
    // 获取 Gd<Node3D>.
    let child = self.base().get_node_as::<Node3D>("Child");
    ```

2. 加载场景并将其实例化为 `RigidBody2D`:
    ```rust
    // mob_scene 声明为类型 Gd<PackedScene> 的字段。
    self.mob_scene = load("res://Mob.tscn");
    
    // instanced 的类型为 Gd<RigidBody2D>。
    let mut instanced = self.mob_scene.instantiate_as::<RigidBody2D>();
    ```

3. 自定义类中 传递了`Node3D` 的信号 `body_entered` 的信号处理函数：
    ```rust
    #[godot_api]
    impl Player {
        #[func]
        fn on_body_entered(&mut self, body: Gd<Node3D>) {
            // body 保存触发信号的 Node3D 对象的引用。
        }
    }
    ```


## 对象管理和生命周期

在处理 Godot 对象时，了解它们的生命周期以及它们何时被销毁是非常重要的。


### 构造

并非所有 Godot 类都能构造；例如，单例（singletons）并没有提供构造函数。

对于其他类，构造函数的名称取决于该类的内存管理方式：

- 对于引用计数的类，构造函数名为 `new_gd`（例如 `TcpServer::new_gd()`）。
- 对于手动管理的类，构造函数名为 `new_alloc（`例如 `Node2D::new_alloc()`）。

[`new_gd()`][api-newgd] 和 [`new_alloc()`][api-newalloc] 函数分别通过扩展traits `NewGd` 和 `NewAlloc` 导入。
它们总是返回类型 `Gd<Self>`。如果你在类名后输入 `::`，IDE 应该会建议正确的构造函数。


### 实例 API

一旦 Godot 对象被创建，你就可以访问它们与引擎交互。

查询和管理对象生命周期的功能直接可用在 `Gd<T>` 类型上。示例如下：

- `instance_id()`  获取 Godot 的对象 ID。
- `clone()` 创建对同一对象的新引用。
- `free()` 手动销毁对象。
- `==` 和 `!=` 用于比较对象的身份。


### 类型转换

如果对象存在继承关系，你可以进行向上或向下转换。gdext 会静态确保转换是合理的。

向下转换使用 `cast::<U>()`，如果转换失败，方法会 panic。你也可以使用 `try_cast::<U>()` 返回一个 `Result`。

```rust
let node: Gd<Node> = ...;

// 我知道这个向下转换一定成功" -> 使用 cast()。
let node2d = node.cast::<Node2D>();
// 替代语法：
let node2d: Gd<Node2D> = node.cast();

// 可失败的向下转换 -> 使用 try_cast()。
let sprite = node.try_cast::<Sprite2D>();
match sprite {
    Ok(sprite) => { /* 访问转换后的 Gd<Sprite2D> */ },
    Err(node) => { /* 访问之前的 Gd<Node> */ },
}
```

向上转换总是无误的。你可以使用  `upcast::<U>()` 消耗值。

```rust
let node2d: Gd<Node2D> = ...;
let node = node2d.upcast::<Node>();
// or, equivalent:
let node: Gd<Node> = node2d.upcast();
```

如果你只需要引用，可以使用 `upcast_ref()` 或 `upcast_mut()`.

```rust
let node2d: Gd<Node2D> = ...;
let node: &Node = node2d.upcast_ref();

let mut refc: Gd<RefCounted> = ...;
let obj: &mut Object = refc.upcast_mut();
```


### 销毁

通过 `new_gd()` 实例化的引用计数类会在最后一个引用超出作用域时自动销毁。
这包括已与 Godot 引擎共享的引用（例如，由 GDScript 代码持有）。

通过 `new_alloc()` 实例化的类需要手动内存管理。这意味着你必须显式调用
[`Gd::free()`][api-gd-free] 或使用像 `Node::queue_free()` 这样的 Godot 方法来处理销毁。


```admonish tip title="关于销毁对象的安全性"

访问销毁的对象是 Godot 中常见的 bug 来源，有时可能会导致未定义行为（UB）。
但在 godot-rust 中并非如此！我们设计了 `Gd<T>` 类型，即使在出现错误时也能保持安全。

如果你尝试访问已销毁的对象，Rust 代码将会 panic。虽然也有 API 可以查询对象是否有效，但我们
通常推荐修复 bug，而不是依赖防御性编程。

```


## 结论

对象是 Rust 绑定中的核心概念。它们表示 Godot 类的实例，无论是引擎类还是用户定义类。
我们已经看到如何构造、管理和销毁这些对象。

但是我们仍然需要 _使用_ 对象，即访问它们类暴露的功能。下一章将深入探讨如何调用 Godot 函数。


[api-gd-free]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html#method.free
[api-gd-from-init-fn]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html#method.from_init_fn
[api-gd]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html
[api-newalloc]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.NewAlloc.html
[api-newgd]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.NewGd.html
[issue-traits]: https://github.com/godot-rust/gdext/issues/426
