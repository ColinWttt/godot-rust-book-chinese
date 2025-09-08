<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 注册常量

常量可以用来将不可变值从 Rust 代码传递到 Godot 引擎中。

另请参见 [GDScript 常量参考][godot-gdscript-constants]。


## 常量声明

在 Rust 中，常量通过 `const` 项声明，并位于类的固有 `impl` 块中。
不能使用 `static` 声明。

`#[constant]` 属性使其在 Godot 中可用。


```rust
#[godot_api]
impl Monster {
    #[constant]
    const DEFAULT_HP: i32 = 100;

    #[func]
    fn from_name_hp(name: GString, hitpoints: i32) -> Gd<Self> { ... }
}
```

在 GDScript 中的使用方式如下：

```php
var nom = Monster.from_name_hp("Nomster", Monster.DEFAULT_HP)
var orc = Monster.from_name_hp("Orc", 200)
```

（这个例子在默认参数实现后可能更适用于默认参数，但它阐明了重点。）


## 限制

Godot 仅支持通过 GDExtension API 注册 **整数常量**。

你可以通过注册一个静态函数来绕过这个限制，在 GDScript 中调用时为 `Monster.DEFAULT_NAME()`。

```rust
#[godot_api]
impl Monster {
    #[func(rename = "DEFAULT_NAME")]
    fn default_name() -> GString {
        "Monster_001".into()
    }
}
```

虽然从技术上讲，你可以使用只读属性，但这会带来问题：

- 你需要已有的类实例。
- 每个对象都会为这个常量占用空间。¹

额外的 `()` 并不会破坏你的游戏。如果你有不想重复计算的值（例如数组），总是可以在 Rust 中用 `thread_local!` 存储。


[godot-gdscript-constants]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#constants
[issue-1151]: https://github.com/godot-rust/gdext/issues/1151

<br>

---

**脚注**

[^zst-properties]: 将来我们可能会有不占用空间的属性，参见 [#1151][issue-1151].
