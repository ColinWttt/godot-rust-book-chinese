<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 引擎单例

理解 [单例模式][singleton]  对于正确使用此系统非常重要。


```admonish info title="争议"
“单例模式”通常被视为一种反模式，因为它违反了多种整洁、模块化代码的良好实践。然而，它也是一个可以用来解决某些设计问题的工具。因此，Godot 内部使用了单例模式，并且 godot-rust 用户也可以使用。

关于批评的更多内容，请参考 [这里][singleton-crit]。

```

引擎单例通过[`godot::classes::Engine`][api-class-engine]注册。

Godot 中的自定义引擎单例：

- 是 `Object` 类型
- 始终可以在 GDScript 和 GDExtension 中访问
- 必须在 `InitLevel::Scene` 阶段手动注册和注销

Godot 在其 API 中提供了 _许多_ 内置单例。您可以在 [这里][godot-singleton-list] 找到完整列表。


[api-class-engine]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Engine.html
[godot-singleton-list]: https://docs.godotengine.org/en/stable/classes/class_@globalscope.html#properties
[singleton-crit]: https://en.wikipedia.org/wiki/Singleton_pattern#Criticism
[singleton]: https://en.wikipedia.org/wiki/Singleton_pattern


## 目录

<!-- toc -->


## 定义单例

定义单例与注册自定义类相同。

```rust
#[derive(GodotClass)]
#[class(init, base=Object)]
struct MyEditorSingleton {
    base: Base<Object>,
}

#[godot_api]
impl MyEditorSingleton {
    #[func]
    fn foo(&mut self) {}
}
```


## 注册单例

单例注册是在初始化的 `InitLevel::Scene` 阶段完成的。

为此，我们可以通过复写 `ExtensionLibrary` trait 方法来自定义初始化/关闭例程。


```rust
struct MyExtension;

#[gdextension]
unsafe impl ExtensionLibrary for MyExtension {
    fn on_level_init(level: InitLevel) {
        if level == InitLevel::Scene {
            // `&str` 用于标识您的单例，稍后可以用它来访问单例。
            Engine::singleton().register_singleton(
                &MyEngineSingleton::class_name().to_string_name(),
                &MyEngineSingleton::new_alloc(),
            );
        }
    }

    fn on_level_deinit(level: InitLevel) {
        if level == InitLevel::Scene {
            // 保留我们的引擎单例实例 和 MyEngineSingleton名称 变量。
            let mut engine = Engine::singleton();
            let singleton_name = &MyEngineSingleton::class_name().to_string_name();


            // 这里，我们手动检索已注册的单例，
            // 以便注销它们并释放内存 —— 注销单例不会自动由库处理。
            if let Some(my_singleton) = engine.get_singleton(singleton_name) {
                // 注销 Godot 中的单例并释放内存，以避免内存泄漏、警告和热重载问题。
                engine.unregister_singleton(singleton_name);
                my_singleton.free();
            } else {
                // 在这里，您可以选择恢复或触发 panic。
                godot_error!("Failed to get singleton");
            }
        }
    }
}
```

```admonish warning title="继承自*RefCounted*的单例"
使用手动管理的类作为自定义单例的基类（通常 `Object` 就足够了），可以避免过早地释放对象。
如果由于某些原因需要将引用计数对象的实例注册为单例，您可以参考这个 [issue thread][refcounted-singleton-issue]，它提供了一些可能的解决方法。
```

[refcounted-singleton-issue]: https://github.com/godot-rust/gdext/issues/522


## 从 GDScript 调用

现在您的单例已可用（并且在重新编译和重新加载后），您应该能够从 GDScript 中像这样访问它：

```GDScript
extends Node

func _ready() -> void:
    MyEditorSingleton.foo()
```


## 从 Rust 调用

您可能也希望从 Rust 中访问您的单例。

```rust
godot::classes::Engine::singleton()
    .get_singleton(StringName::from("MyEditorSingleton"));
```

有关此方法的更多信息，请参阅 [API 文档][method-get-singleton]。

[method-get-singleton]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Engine.html#method.get_singleton


## 单例和 `SceneTree`


单例不能安全地访问场景树。在任何给定时刻，它们可能存在而没有活动的场景树。

虽然技术上可以通过一些取巧的方法访问场景树，但 **强烈建议** 为此目的使用自定义的 `EditorPlugin`。
创建一个 `EditorPlugin` 允许注册一个“自动加载的单例”，这个单例是一个 `Node`（或其派生类型），并且当游戏启动时，Godot 会自动将其加载到 `SceneTree` 中。
