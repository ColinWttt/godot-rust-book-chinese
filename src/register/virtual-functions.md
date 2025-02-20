<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 脚本复写Rust定义的虚函数(script-virtual)

GDExtension API 允许您在 Rust 中定义虚函数，这些函数可以在附加到您的对象上的脚本中被复写。

请注意，这些函数在概念上与如 `ready()` 这样的虚函数不同，`ready()` 是 _由Godot_ 定义并 _由您_（在 Rust 中）重写的。

因此，特别强调“script-virtual”。


```admonish note title="兼容性"
此功能从 Godot 4.3 版本开始提供。

<sub>（包括2024年2月13日之后的开发版和nightly版）。</sub>

```


## 目录

<!-- toc -->


## 一个很好的例子

以我们的 `Monster`为例，假设我们有不同类型的怪物，并且希望自定义它们的行为。我们可以在 Rust 中编写所有怪物共有的逻辑，并使用 GDScript 快速原型化特定部分。

例如，我们可以尝试两个怪物：`Orc` 和 `Goblin`。每个怪物都有不同的行为，这些行为被编码在各自的 GDScript 文件中。项目结构可能如下所示：

```txt
project_dir/
│
├── godot/
│   ├── .godot/
│   ├── project.godot
│   ├── MonsterGame.gdextension
│   └── Scenes
│       ├── Monster.tscn
│       ├── Orc.gd
│       └── Goblin.gd
│
└── rust/
    ├── Cargo.toml
    └── src/
        ├── lib.rs
        └── monster.rs
```

 `Monster.tscn`编码了一个简单的场景，根节点是 `Monster`（我们的 Rust 类，继承自 `Node3D`）。此节点将是附加脚本的对象。


## 步骤说明


### Rust 默认行为

让我们从这个类的定义开始：

```rust
use godot::prelude::*;

#[derive(GodotClass)]
#[class(init, base=Node3D)]
struct Monster {
    base: Base<Node3D>
}
```

现在，我们可以实现一个 Rust 函数来计算怪物每次攻击造成的伤害。传统上，我们会这样编写：


```rust
#[godot_api]
impl Monster {
    #[func]
    fn damage(&self) -> i32 {
        10
    }
}
```

该方法无论如何，始终返回 `10`。为了在附加到 `Monster` 节点的脚本中自定义此行为，我们可以在 Rust 中定义一个 _虚方法_，该方法可以在 GDScript 中 _复写_。这里的Rust 代码被称为 _默认_ 实现。

```admonish note title="前期与后期绑定"
_虚（virtual）_（也称为 _后期绑定_）意味着涉及动态分发：实际调用的方法是在运行时确定的，具体取决于是否有脚本附加到 `Monster` 节点 — 如果有，具体是哪个脚本。

这与 _前期绑定_ 相对，前期绑定是在编译时通过静态分发解决的。
```

虽然传统 Rust 可能使用 trait 对象（`dyn Trait`）来实现后期绑定，但 godot-rust 提供了更直接的方法。

使方法成为虚方法非常简单：只需在 `#[func]` 属性中添加 `virtual` 关键字。


```rust
#[godot_api]
impl Monster {
    #[func(virtual)]
    fn damage(&self) -> i32 {
        10
    }
}
```

就是这么简单。现在，您的怪物可以在脚本中自定义。


### 在 GDScript 中复写

在 GDScript 文件中，您现在可以复写 Rust 的 `damage` 方法为 `_damage`。方法前缀加上下划线，这是 Godot 中虚方法（如 `_ready` 或 `_process`）的命名约定。

以下是 `Orc` 和 `Goblin` 脚本的示例：

```php
# Orc.gd
extends Monster

func _damage():
    return 20
```

```php
# Goblin.gd
extends Monster

# 随机伤害，范围为 5 到 15。
# 类型注解是可选的。
func _damage() -> int:
    return randi() % 11 + 5
```

如果您的 GDScript 中的签名与 Rust 签名不匹配，Godot 会产生错误。


### 动态行为

现在，让我们在 Rust 代码中调用  `damage()`：

```rust
fn monster_attacks_player(monster: Gd<Monster>, player: Gd<Player>) {
    // 计算伤害。
    let damage_points: i32 = monster.bind().damage();

    // 将伤害应用到玩家。
    player.bind_mut().take_damage(damage_points);
}
```

在上述示例中，`damage_points` 的值是多少？

答案取决于具体情况：

- 如果`Monster`节点没有附加脚本，`damage_points` 将是 `10`（Rust 中的默认实现）。
- 如果`Monster`节点附加了 `Orc.gd` 脚本，`damage_points` 将是 `20`。
- 如果`Monster`节点附加了 `Goblin.gd` 脚本，`damage_points` 将是一个在 `5` 到 `15` 之间的随机数。


## 权衡取舍

你可能会问：如果只需要计算一个简单的伤害数值，为什么不使用一个简单的 `match` 语句？

你说得对，如果只需要Rust 中 `match` 就能满足你的需求，那么直接使用它即可。然而，基于脚本的方法有一些优势，尤其是在处理比单一伤害值计算更复杂的场景时：

- 你可以准备多种脚本，来处理不同的行为，例如不同的关卡或敌人 AI 行为。在 Godot 编辑器中，你可以根据需要轻松地切换脚本，或者让不同的 `Monster` 实例使用不同的脚本进行对比。
- 切换行为不需要重新编译 Rust 代码。如果你与不太熟悉 Rust 的游戏设计师、模组制作者或艺术家合作，但他们仍然希望进行实验，这会非常有用。

也就是说，如果你的编译时间较短（gdext 本身非常轻量），并且更喜欢将逻辑放在 Rust 中，这也是一个有效的选择。为了保留快速切换行为的选项，你可以使用 `#[export]` 导出的枚举来选择行为，然后在 Rust 中进行调度。

最终，`#[func(virtual)]` 只是 godot-rust 提供的多种抽象机制中的一个额外工具。由于 Godot 的范式主要围绕将脚本附加到节点上展开，因此该功能与引擎非常契合。


## 局限性

```admonish warning title="警告"
Godot 的脚本虚函数与面向对象编程中的虚函数在各方面的行为不同。
请确保理解其局限性。
```

与面向对象语言中的虚方法（如 C++、C#、Java、Kotlin、PHP 等）相比，有一些重要的区别需要注意。


1. **默认实现无法从 Godot 访问**

    在 Rust 中，调用 `monster.bind().damage()` 会自动查找脚本复写，并在没有脚本附加时回退到 Rust 默认实现。然而，在 GDScript 中，您无法调用默认实现。
    调用 `monster._damage()` 在没有脚本的情况下会失败。Rust 的反射调用（例如 `Object::call()`）也是如此。

    下划线 `_` 前缀的意义在于：理想情况下，您不应直接从脚本调用虚函数。

    为了解决这个问题，您可以在 Rust 中声明一个单独的 `#[func] fn default_damage()`，该函数将作为常规方法注册，因此可以从脚本中调用。
    为了保留 Rust 的便捷回退行为，您可以在 Rust 的 `damage()` 方法中调用 `default_damage()`。

2. **无法访问 `super` 方法。**

    在面向对象语言中，您可以从复写的方法中调用基类方法，通常使用 `super` 或 `base` 关键字。

    由于第 1 点的原因，这个默认方法在脚本中无法访问。然而，可以使用相同的解决方法。

3. **有限的重入性。**

    如果您从 Rust 调用虚方法，它可能会调度到脚本实现。Rust 端持有对象的共享引用（`&self`）或独占引用（`&mut self`）——即隐式的 `Gd::bind()` 或 `Gd::bind_mut()`guard。
    如果脚本实现随后访问相同的对象（例如通过设置 `#[var]` 属性），由于双重借用错误可能会发生 `panic`。

    目前，您可以通过在方法上使用 `#[func(gd_self, virtual)]` 来绕过此问题。`gd_self` 要求第一个参数为 `Gd<Self>` 类型，这避免了调用 `bind`，因此避免了借用问题。

我们正在观察社区如何使用虚函数，并计划在可能的情况下缓解这些限制。如果您有任何建议，欢迎与我们分享！


## 脚本类型

虽然本页重点讨论 GDScript，Godot 还提供了其他脚本功能。值得注意的是，您可以使用 [C# 脚本][godot-csharp]，如果您使用 Mono 运行时来运行 Godot。

该库还提供了一个专用的 trait [`ScriptInstance`][api-scriptinstance]，允许用户提供基于 Rust 的“脚本”。有关详细信息，请查阅其文档。

您还可以使用 [`classes::Script`][api-class-script] API 及其继承类（如 [`classes::GDScript`][api-class-gdscript]）完全以编程方式配置脚本。这通常会违背脚本的初衷，但在这里提及以供参考。


## 结论

在本章中，我们展示了如何在 Rust 中定义虚函数，并如何在 GDScript 中复写它们。这为两种语言之间提供了一个额外的集成层，并允许在编辑器中轻松实验可交换的行为。

[api-class-gdscript]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.GDScript.html
[api-class-script]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Script.html
[api-scriptinstance]: https://godot-rust.github.io/docs/gdext/master/godot/obj/script/trait.ScriptInstance.html
[godot-csharp]: https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/index.html
