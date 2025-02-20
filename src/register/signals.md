<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 注册信号

目前，gdext对信号的支持非常有限，通过 `#[signal]` 属性实现。有关详细信息，请查阅其[API文档][api-signal]。

信号的注册将在未来完全重构，并且会有破坏性的API变更。

作为替代方法，你可以使用Godot的动态API来注册信号。[`Object类`][api-object]具有`connect()`和`emit_signal()`方法，分别可以用来连接和发射信号。

请看[GDScript中信号的参考][godot-gdscript-signals]。


[api-object]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Object.html
[api-signal]: https://godot-rust.github.io/docs/gdext/master/godot/register/derive.GodotClass.html#signals
[godot-gdscript-signals]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#signals
