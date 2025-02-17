<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 构造函数（Constructors）

While Rust does not have constructors as a language feature (like C++ or C#), associated functions that return a new object are commonly
called "constructors". We extend the term to include slightly deviating signatures, but conceptually _constructors_ are always
used to construct new objects.

Godot has a special constructor, which we call the _Godot default constructor_ or simply `init`. This is comparable to the `_init` method in
GDScript.


## 目录

<!-- toc -->


## 默认构造函数

在 gdext 中，任何 `GodotClass` 对象的构造函数被称为 `init`。这个构造函数是用来在 Godot 中实例化对象的。当场景树需要实例化对象，或者当你在 GDScript 中调用 `Monster.new()` 时，它会被调用。

定义构造函数有两种选择：可以让 gdext 自动生成，也可以手动定义。如果你不需要 Godot 默认构造你的对象，还可以选择不使用 `init`。


### 库生成的 `init`

你可以使用 `#[class(init)]` 来为你生成一个构造函数。这仅限于简单的情况，它会对每个字段调用  `Default::default()`  (除了 `Base<T>` , 后者会正确地与基类对象连接)。


```rust
#[derive(GodotClass)]
#[class(init, base=Node3D)]
struct Monster {
    name: String,          // initialized to ""
    hitpoints: i32,        // initialized to 0
    base: Base<Node3D>,    // wired up
}
```

如果需要为字段提供其他默认值，可以使用 `#[init(val = value)]`。此方法仅适用于简单场景，因为它可能会导致代码难以阅读和难以理解的错误信息。此外，此 API 仍可能发生变更。

```rust
#[derive(GodotClass)]
#[class(init, base=Node3D)]
struct Monster {
    name: String,          // initialized to ""
   
    #[init(val = 100)]
    hitpoints: i32,        // initialized to 100
    
    base: Base<Node3D>,    // wired up
}
```


### 手动定义 `init`

我们可以通过复写特征的关联函数 `init` 来提供手动定义的构造函数：

```rust
#[derive(GodotClass)]
#[class(base=Node3D)] // No init here, since we define it ourselves.
struct Monster {
    name: String,
    hitpoints: i32,
    base: Base<Node3D>,
}

#[godot_api]
impl INode3D for Monster {
    fn init(base: Base<Node3D>) -> Self {
        Self {
            name: "Nomster".to_string(),
            hitpoints: 100,
            base,
        }
    }
}
```

如你所见，`init`  函数接受一个 `Base<Node3D>` 作为唯一参数。这个是基类实例，通常只是被转发到结构体中的相应字段（这里是 `base`）。

`init` 方法总是返回 `Self`。你可能注意到，目前这是构造 `Monster` 实例的唯一方式。一旦你的结构体包含了一个基类字段，你就不能再提供自己的构造函数了，因为你不能为这个字段提供一个值。这是设计上的原因，并且确保了 _如果_ 你需要访问基类，这个基类是直接来自 Godot 的。

不过，不用担心：你仍然可以提供各种构造函数，只是它们需要通过专门的函数来实现，这些函数内部调用 `init`。更多内容将在下一节讨论。


### 禁用 `init`

你并不总是需要为 Godot 提供默认构造函数。没有构造函数的原因包括：

- 你的类不是一个应该作为场景文件一部分添加到树中的节点。
- 你需要为对象的不可变状态提供自定义参数 —— 默认值没有意义。
- 你只需要从 Rust 代码中构造对象，而不是从 GDScript 或 Godot 编辑器中构造。
  
要禁用 `init` 构造函数，可以使用 `#[class(no_init)]`：

```rust
#[derive(GodotClass)]
#[class(no_init, base=Node3D)]
struct Monster {
    name: String,
    hitpoints: i32,
    base: Base<Node3D>,
}
```

如果不提供/生成 `init` 方法并且忘记使用 `#[class(no_init)]`，将导致编译时错误。


## 自定义构造函数

默认的构造函数 `init` 并不总是有用，因为它可能会让对象处于不正确的状态。

例如， `Monster` 在构造时总会有相同的 `name` 和 `hitpoints` 这可能不是我们希望的。让我们提供一个更合适的构造函数，它将这些属性作为参数。

```rust
// Default constructor from before.
#[godot_api]
impl INode3D for Monster {
    fn init(base: Base<Node3D>) -> Self { ... }
}

// New custom constructor.
#[godot_api]
impl Monster {
    #[func] // Note: the following is incorrect.
    fn from_name_hp(name: GString, hitpoints: i32) -> Self { 
        ...
    }
}
```

但现在，如何填补空白呢？`Self`需要一个基类对象，我们该如何获取它呢？实际上，我们不能在这里返回 `Self`。

```admonish info title="传递对象"
在 Rust 与 Godot 交互时，所有对象（类实例）都需要通过 `Gd` 智能指针传递——无论它们是作为参数还是返回类型。

`init` 和一些其他 gdext 提供的函数的返回类型是一个例外，因为库在此时要求你拥有一个原始对象的_值_。你在自己定义的 `#[func]` 函数中不需要返回 `Self`。

详细信息请参见 [关于对象的章节][book-objects] 或 [`Gd<T>` API 文档][api-gd]。

```

所以我们需要返回  `Gd<Self>` 而不是 `Self`.


### 带有基类字段的对象

如果你的类  `T` 包含一个 `Base<...>` 字段，你不能创建一个独立的实例——你必须将其封装在`Gd<T>`。
你也不能再从 `Gd<T>` 智能指针中提取 `T`；因为它可能已经与 Godot 引擎共享，这样做将不是一个安全的操作。

为了构造  `Gd<Self>`, 我们可以使用 [`Gd::from_init_fn()`][api-gd-from-init-fn], 它接受一个闭包。这个闭包接受一个 `Base` 对象并返回 `Self` 的实例。
换句话说，它的签名与 `init` 相同——这提供了一种替代的构造 Godot 对象的方法，同时允许额外的上下文传递。

`Gd::from_init_fn()` 的结果是一个 `Gd<Self>` 对象，它可以直接由 `Monster::from_name_hp()` 返回。

```rust
#[godot_api]
impl Monster {
    #[func]
    fn from_name_hp(name: GString, hitpoints: i32) -> Gd<Self> {
        // Function contains a single statement, the `Gd::from_init_fn()` call.
        
        Gd::from_init_fn(|base| {
            // Accept a base of type Base<Node3D> and directly forward it.
            Self {
                name: name.into(), // Convert GString -> String.
                hitpoints,
                base,
            }
        })
    }
}
```

就是这样！刚刚添加的关联函数现在已在 GDScript 中注册，并有效地作为构造函数使用：

```php
var monster = Monster.from_name_hp("Nomster", 100)
```


### 没有基类字段的对象

对于没有基类字段的类，你可以简单地使用 [`Gd::from_object()`][api-gd-from-object]，而不是 `Gd::from_init_fn()`。

这通常对于 _数据包_ 很有用，数据包不定义太多逻辑，但它是一种面向对象的方式，将相关数据打包在单一类型中。这些类通常是 `RefCounted` 或 `Resource` 的子类。

```rust
#[derive(GodotClass)]
#[class(no_init)] // We only provide a custom constructor.
// Since there is no #[class(base)] key, the base class will default to RefCounted.
struct MonsterConfig {
    color: Color,
    max_hp: i32,
    tex_coords: Vector2i,
}

#[godot_api]
impl MonsterConfig {
    // Not named 'new' since MonsterConfig.new() in GDScript refers to default. 
    #[func] 
    fn create(color: Color, max_hp: i32, tex_coords: Vector2i) -> Gd<Self> {
        Gd::from_object(Self {
            color,
            max_hp,
            tex_coords,
        })
    }
}
```


## 析构函数

如果你通过 [RAII][wiki-raii] 管理内存，通常你不需要声明自己的析构函数。但是如果你确实需要自定义的清理逻辑，只需为你的类型声明 `Drop` 特征：

```rust
impl Drop for Monster {
    fn drop(&mut self) {
        godot_print!("Monster '{}' is being destroyed!", self.name);
    }
}
```

当 Godot 命令销毁你的 `Gd<T>` 智能指针时——无论是手动释放，还是最后一个引用超出作用域——都会调用`Drop::drop()`。


## 结论

构造函数允许以各种方式初始化 Rust 类。你可以生成、实现或禁用默认构造函数 `init`，并且可以提供任意多的具有不同签名的自定义构造函数。

[api-gd-from-init-fn]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html#method.from_init_fn
[api-gd-from-object]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html#method.from_object
[api-gd]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html
[book-objects]: ../intro/objects.md
[wiki-raii]: https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization
