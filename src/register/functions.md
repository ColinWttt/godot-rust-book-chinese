<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 注册函数


函数是任何编程语言中执行逻辑的基本组成部分。gdext 库允许你注册函数，这样就可以从 Godot 引擎和 GDScript 中调用它们。

函数的注册始终发生在标注了 `#[godot_api]` 的 impl 块内。

另请参见 [GDScript 函数参考][godot-gdscript-functions]。


## 目录

<!-- toc -->


## Godot 特殊函数

```admonish info title="接口 traits"
每个引擎类都有一个相关的trait，它的名称与类相同，但前面加上了字母 `I`，表示“接口”。

该特征没有必需的函数，但你可以重写任何函数来定制对象对 Godot 的行为。

对于这个tra的任何 `impl` 块都必须使用 `#[godot_api]` 属性宏进行标注。
```

```admonish info title="godot_api 宏"
属性过程宏 `#[godot_api]` 应用于 `impl` 块，并将其中的项标记为注册对象。

它不接受任何参数。

详细信息请参阅 [API 文档][api-godot-api]。
```

由接口trait（以 `I` 开头）提供的函数称为 _Godot 特殊函数_。这些函数可以被重写，并允许你影响对象的行为。最常见的是对象的 _生命周期_ 钩子，定义一些在特定事件（如创建、进入场景树或每帧更新）时执行的逻辑。

在我们的例子中，`Node3D` 提供了 `INode3D` trait。

以下是其生命周期方法的一小部分示例。完整列表请参见 [`INode3D` 文档][api-inode3d]。


```rust
#[godot_api]
impl INode3D for Monster {
    // 实例化对象。
    fn init(base: Base<Node3D>) -> Self { ... }
    
    // 节点在场景树准备好时调用。
    fn ready(&mut self) { ... }
    
    // 每帧调用。
    fn process(&mut self, delta: f64) { ... }
    
    // 每物理帧调用。
    fn physics_process(&mut self, delta: f64) { ... }
    
    // 对象的字符串表示。
    fn to_string(&self) -> GString { ... }

    // 处理用户输入。
    fn input(&mut self, event: Gd<InputEvent>) { ... }

    // 处理生命周期通知。
    fn on_notification(&mut self, what: Node3DNotification) { ... }
}
```

如你所见，一些方法使用 `&mut self`，而一些方法使用 `&self`，这取决于它们是否通常会修改对象。某些方法还有返回值，这些值会被传回引擎。例如，`to_string()` 返回的 `GString` 会在 GDScript 中打印对象时使用。

让我们实现 `to_string()`，再次展示类定义以供快速参考。


```rust
#[derive(GodotClass)]
#[class(base=Node3D)]
struct Monster {
    name: String,
    hitpoints: i32,
    
    base: Base<Node3D>,
}

#[godot_api]
impl INode3D for Monster {      
    fn to_string(&self) -> GString {
        let Self { name, hitpoints, .. } = &self;
        format!("Monster(name={name}, hp={hitpoints})").into()
    }
}
```


## 用户定义的函数


### 方法


除了 Godot 特殊函数外，你还可以注册自己的函数。你需要在一个固有的 `impl` 块中声明它们，并同样用 `#[godot_api]` 注解。

每个函数需要一个 `#[func]` 属性来将其注册到 Godot。如果省略 `#[func]`，该函数只能在 Rust 代码中可见。

让我们给 `Monster` 类添加两个方法：一个对其造成伤害，另一个返回其名称。

```rust
#[godot_api]
impl Monster {
    #[func]
    fn damage(&mut self, amount: i32) {
        self.hitpoints -= amount;
    }
    
    #[func]
    fn get_name(&self) -> GString {
        self.name.clone()
    }
}
```

上面这些方法现在可以在 GDScript 中使用。你可以这样调用它们：

```GDScript
var monster = Monster.new()
# ...
monster.damage(10)
print("A monster called ", monster.get_name())
```

如你所见，Rust 类型会自动映射到它们在 GDScript 中的对应类型。在这个例子中，`i32` 会变成 `int`，`GString` 会变成 `String`。有时会有多个映射方式，例如 Rust 的 `u16` 也会映射为 GDScript 中的 `int`。


### 关联函数

除了 **方法**（有y `&self` 或 `&mut self`），你还可以注册 **关联函数**（没有接收者）。在 GDScript 中，后者称为 “静态函数”。

例如，我们可以添加一个关联函数，用来生成一个随机的怪物名称：

```rust
#[godot_api]
impl Monster {
    #[func]
    fn random_name() -> GString {
        // ...
    }
}
```

然后可以在 GDScript 中这样调用它：

```GDScript
var name: String = Monster.random_name()
```

当然，也可以声明参数。

关联函数有时对用户定义的构造函数非常有用，正如我们将在下一章中看到的。


## 方法与对象访问

当你定义自己的 Rust 函数时，有两种使用场景非常常见：

- 你想通过 `Gd` 指针从外部调用你的 Rust 方法。
- 你想访问基类的方法（例如 `Node3D`）。

本节将解释如何做到这两点。


### 调用 Rust 方法（binds）

如果你现在有一个 `monster: Gd<Monster>`，它存储的是上面定义的 `Monster` 对象，你不能直接调用 `monster.damage(123)`。
Rust 比 C++ 更严格，要求在任何时刻只能有一个 `&mut Monster`引用。由于 `Gd` 指针可以自由克隆，直接通过 `DerefMut` 访问不足以保证不产生别名（non-aliasing）。

为了解决这个问题，godot-rust 使用了内部可变性模式，类似于 [`RefCell`][rust-refcell] 的工作方式。

简而言之，当你需要共享（不可变）访问一个 Rust 对象时，从 `Gd` 指针中使用 [`Gd::bind()`][api-gd-bind]。
当你需要独占（可变）访问时，使用 [`Gd::bind_mut()`][api-gd-bindmut]。


```rust
let monster: Gd<Monster> = ...;

// 使用 bind() 进行不可变访问：
let name: GString = monster.bind().get_name();

// 使用 bind_mut() 进行可变访问 — 我们首先重新绑定对象：
let mut monster = monster;
monster.bind_mut().damage(123);
```

Rust 的常规可见性规则适用：如果你的函数应该在另一个模块中可见，请声明为 `pub` 或 `pub(crate)`。


```admonish note title="#[func]的必要性"
`#[func]` 属性 _仅_ 使函数在 Godot 引擎中可用。它与 Rust 的可见性（`pub`、`pub(crate)` 等）是独立的，不会影响是否可以通过 `Gd::bind()` 和 `Gd::bind_mut()` 访问方法。

如果你只需要在 Rust 中调用一个函数，可以不加 `#[func]` 注解。你可以稍后再添加它。

```

`bind()` 和 `bind_mut()` 返回 _guard 对象_。在运行时，库会验证借用规则是否得到遵守，否则会 panic。

通过多个语句重用 guards 是有好处的，但要确保保持其作用域的限制，以免不必要地限制对对象的访问（尤其是使用 bind_mut() 时）。


```rust
fn apply_monster_damage(mut monster: Gd<Monster>, raw_damage: i32) {
    // Artificial scope:
    {
        let guard = monster.bind_mut(); // locks object -->
        let armor = guard.get_armor_multiplier();
        
        let damage = (raw_damage as f32 * armor) as i32;

        guard.damage(damage)
    } // <-- until here, where guard lifetime ends.

    // Now you can pass the pointer on to other routines again.
    check_if_dead(monster);
}
```


### 从 `self` 访问基类

在一个类中，你没有直接指向当前实例的 `Gd<T>` 指针来访问基类的方法。所以你不能像在 [_调用函数_ 章节][book-godot-api-functions] 中那样，直接使用 `gd.set_position(...)` 等方法。

但是，你可以通过 [`base()` 和 `base_mut()`][api-withbasefield-base] 访问基类 API。这要求你的类定义一个 `Base<T>` 字段。假设我们添加了一个 `velocity`字段和两个新方法：

```rust
#[derive(GodotClass)]
#[class(base=Node3D)]
struct Monster {
    // ...
    velocity: Vector2,
    base: Base<Node3D>,
}

#[godot_api]
impl Monster {
    pub fn apply_movement(&mut self, delta: f32) {
        // 只读访问：
        let pos = self.base().get_position();
      
        // 写入访问 (mutating 方法):
        self.base_mut().set_position(pos + self.velocity * delta)
    }

    // 此方法仅具有只读访问（&self）。
    pub fn is_inside_area(&self, rect: Rect2) -> String 
    {
         // 这里只能调用 base()，不能调用 base_mut()。
        let node_name = self.base().get_name();
        
        format!("Monster(name={}, velocity={})", node_name, self.velocity)
    }
}
```

`base()` 和 `base_mut()` 都在扩展 trait [`WithBaseField`][api-withbasefield] 中定义。它们返回 _guard 对象_，这些对象会根据 Rust 的借用规则防止对 `self` 的其他访问。
你可以在多个语句之间重用 guard，但需要确保将其作用域限制在最小范围，以避免不必要地限制对 `self` 的访问：

```rust
    pub fn apply_movement(&mut self, delta: f32) {
        // 作用域：
        {
            let guard = self.base_mut(); // locks `self` -->
            let pos = guard.get_position();
  
            guard.set_position(pos + self.velocity * delta)
        } // <-- 到这里，guard 生命周期结束。
  
        // 现在可以再次调用其他 `self` 方法。
        self.on_position_updated();
    }
```

当然，你也可以不使用额外的作用域，直接调用 [`drop(guard)`][rust-mem-drop] 来结束guard的生命周期。


```admonish note title="不要将bind/bind_mut 和 base/base_mut组合使用"
代码像 `object.bind().base().some_method()` 不必要地冗长且低效。  
如果你有一个 `Gd<T>` 指针，直接使用 `object.some_method()` 即可。  

将 `bind()`/`bind_mut()`  立即与 `base()`/`base_mut()`组合使用是错误的。后者应该只在类的 `impl` 中调用。

```


### 从内部获取 `Gd<Self>`

在某些情况下，你可能需要获取指向当前实例的 `Gd<T>` 指针。这种情况发生在你想将它传递给其他方法，或者需要将 `self` 的指针存储在数据结构中时。

`WithBaseField` 提供了一个 `to_gd()` 方法，它返回一个具有正确类型的 `Gd<Self>`。

下面是一个例子。`monster` 被传入一个hash map，它可以根据怪物是否存活来注册或注销自己。

```rust
#[godot_api]
impl Monster {
    // 函数根据怪物的状态（是否活着）注册或注销每个怪物。
    fn update_registry(&self, registry: &mut HashMap<String, Gd<Monster>>) {
        if self.is_alive() {
            let self_as_gd: Gd<Self> = self.to_gd();
            registry.insert(self.name.clone(), self_as_gd);
        } else {
            registry.remove(&self.name);
        }
    }
}
```

```admonish warning title="不要在类方法中使用to_gd()后bind"

`base()` 和 `base_mut()` 使用一种巧妙的机制，通过“重新借用”当前对象的引用。
这允许重新进入式调用，例如 `self.base().notify(...)`，可能会调用 `ready(&mut self)`。这里的 `&mut self` 是对调用站点 `self` 的重新借用。

当你使用 `to_gd()` 时，借用检查器会将其视为一个独立的对象。如果你在类的 `impl` 中对它调用 `bind_mut()`，你会立即遇到双重借用 panic。相反，使用 `to_gd()` 获取指针后，直到当前方法结束时再访问。

```


## 结论


本页向你介绍了如何在 Godot 中注册函数：

- 特殊方法，钩入对象的生命周期。
- 用户定义的方法和关联函数，将 Rust API 暴露给 Godot。

它还展示了方法与对象如何交互：通过 `Gd<T>` 调用 Rust 方法，并与基类 API 一起工作。

这些只是一些用例，你在设计 Rust 和 GDScript 之间的接口时非常灵活。在下一页中，我们将探讨一种特殊的函数类型：构造函数。


[api-godot-api]: https://godot-rust.github.io/docs/gdext/master/godot/register/attr.godot_api.html
[api-inode3d]: https://godot-rust.github.io/docs/gdext/master/godot/classes/trait.INode3D.html
[godot-gdscript-functions]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#functions
[api-withbasefield]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html
[api-withbasefield-base]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html#method.base
[rust-refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[rust-mem-drop]: https://doc.rust-lang.org/std/mem/fn.drop.html
[book-godot-api-functions]: ../godot-api/functions.html#godot-functions
[api-gd-bind]: https://godot-rust.github.io/docs/gdext/master/godot/prelude/struct.Gd.html#method.bind
[api-gd-bindmut]: https://godot-rust.github.io/docs/gdext/master/godot/prelude/struct.Gd.html#method.bind_mut
