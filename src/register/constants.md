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

常量在 Rust 中作为 `const` 项声明，放在类的 `impl` 块中。

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


## 静态字段

目前`static` 字段无法作为常量注册。


[godot-gdscript-constants]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#constants
