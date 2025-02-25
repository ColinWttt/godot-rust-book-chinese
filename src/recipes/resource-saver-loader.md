<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# `Resource` 保存器和加载器

[`ResourceFormatSaver`][godot-saver] 和 [`ResourceFormatLoader`][godot-loader] 允许你使用自定义过程序列化和反序列化你派生自 Rust 的`Resource`类，并且定义新的已识别文件扩展名。
如果你有包含 _纯 Rust 状态_ 的资源，这通常非常有用。
在这个上下文中，“纯”是指你的结构体成员中没有任何 `#[var]` 或类似的注解，即 Godot 不知道它们。这在你使用 Rust 库时很容易发生。

以下示例为你提供了一个可以复制粘贴的起点。对于高级用例，请参考这些类的 Godot 文档。

首先，你需要在库的入口点 `InitLevel::Scene` 中调用提供的函数。这可以确保正确初始化和清理你的加载器/保存器。


```rust
// 以下导入需要在接下来的代码示例中使用。
use godot::classes::{
    Engine, IResourceFormatLoader, IResourceFormatSaver, ResourceFormatLoader,
    ResourceFormatSaver, ResourceLoader, ResourceSaver,
};
use godot::prelude::*;

#[gdextension]
unsafe impl ExtensionLibrary for MyGDExtension {
    // 在扩展加载时注册单例。
    fn on_level_init(level: InitLevel) {
        if level == InitLevel::Scene {
            Engine::singleton().register_singleton(
                "MyAssetSingleton",
                &MyAssetSingleton::new_alloc(),
            );
        }
    }

    // 在扩展退出时注销单例。
    fn on_level_init(level: InitLevel) {
        if level == InitLevel::Scene {
            Engine::singleton().unregister_singleton("MyAssetSingleton");
            my_singleton.free();
        }
    }
}
```

定义单例来追踪你的加载器和保存器。

```rust

// 定义单例，包含所有加载器/保存器作为成员，
// 用于保持对象引用在后续销毁。
#[derive(GodotClass)]
#[class(base=Object, tool)]
struct MyAssetSingleton {
    base: Base<Object>,
    loader: Gd<MyAssetLoader>,
    saver: Gd<MyAssetSaver>,
}

#[godot_api]
impl IObject for MyAssetSingleton {
    fn init(base: Base<Object>) -> Self {
        let saver = MyAssetSaver::new_gd();
        let loader = MyAssetLoader::new_gd();

        // 在 Godot 中注册加载器和保存器。
        //
        // 如果你希望默认扩展名是由加载器定义的，
        // 请将 `at_front` 参数设置为 true。否则，你也可以删除该构建器。
        // Godot 目前没有提供完全禁用内置加载器的方法。
        // 警告：如果你有 _纯 Rust 状态_，内置加载器将无法正常工作。

        ResourceSaver::singleton().add_resource_format_saver_ex(&saver)
            .at_front(false)
            .done();
        ResourceLoader::singleton().add_resource_format_loader(&loader);
        
        Self { base, loader, saver }
    }
}
```

```admonish warning title="at_front 行为"
目前在 Godot 中，`at_front` 的顺序可能不会按预期工作。更多信息，请查看 PR godot#101543 或文档讨论 [#65][book#65]。

```

一个最简的**保存器**代码，定义了所有必需的虚方法：

```rust
#[derive(GodotClass)]
#[class(base=ResourceFormatSaver, init, tool)]
struct MyAssetSaver {
    base: Base<ResourceFormatSaver>,
}

#[godot_api]
impl IResourceFormatSaver for MyAssetSaver {
    // 如果你想要自定义扩展名（例如：resource.myextension），
    // 则覆盖此方法。
    fn get_recognized_extensions(
        &self,
        res: Option<Gd<Resource>>
    ) -> PackedStringArray {
        let mut array = PackedStringArray::new();
        
        // 尽管 Godot 文档说明你不需要进行此检查，
        // 但实际上这是必要的。
        if Self::is_recognized_resource(res) {
            // 也可以为每个保存器添加多个扩展名。
            array.push("myextension");
        }
        
        array
    }

    // 所有该保存器应处理的资源类型都必须返回 true。
    fn is_recognized_resource(res: Option<Gd<Resource>>) -> bool {
        // 每个保存器也可以添加多个资源类型。
        res.expect("Godot called this without an input resource?")
            .is_class("MyResourceType")
    }


    // 定义实际保存resource的逻辑。
    fn save(
        &mut self,
        // 当前正在保存的resource。
        resource: Option<Gd<Resource>>,
        // resource将保存到的路径。
        path: GString,
        // 用于保存的flags（见下面的链接）。
        flags: u32,
    ) -> godot::global::Error {
        // 在此处添加保存逻辑，使用 `GFile` API（见下方链接）。
        
        godot::global::Error::OK
    }
}
```

这是 `SaverFlags`（[Godot][godot-saverflags], [Rust][api-saverflags]）和[`GFile`][api-gfile] 的文档链接。

一个最简的**加载器**代码，定义了所有必需的虚方法：

```rust
#[derive(GodotClass)]
#[class(init, tool, base=ResourceFormatLoader)]
struct MyAssetLoader {
    base: Base<ResourceFormatLoader>,
}

#[godot_api]
impl IResourceFormatLoader for MyAssetLoader {
    // 应该在此处添加，你希望的加载器重定向的所有文件扩展名。
    fn get_recognized_extensions(&self) -> PackedStringArray {
        let mut arr = PackedStringArray::new();
        arr.push("myextension");
        arr
    }

    // 该加载器处理的所有resource类型。
    fn handles_type(&self, ty: StringName) -> bool {
        ty == "MyResourceType".into()
    }

    // 应该返回你的 resource的字符串化名称。
    fn get_resource_type(&self, path: GString) -> GString {
        // 在 Godot 中，扩展名参数总是带有 `.`，所以不要忘记这一点 ;)
        if path.get_extension().to_lower() == ".myextension".into() {
            "MyResourceType".into()
        } else {
            // 如果不处理给定的resource，此函数必须返回一个空字符串。
            GString::new()
        }
    }

    // 实际加载和解析你的数据。
    fn load(
        &self,
        // 应该打开的加载资源的路径。
        path: GString,
        // 如果资源是导入步骤的一部分，你可以通过此参数访问原始文件。
        // 否则这个路径等同于普通路径。
        original_path: GString,
        // 如果资源是通过 `load_threaded_request()` 加载的，此参数为 true。
        // Godot 中的内部实现也忽略此参数。
        _use_sub_threads: bool,
        // 如果你希望提供自定义缓存，则此参数为 `CacheMode` 枚举。
        // 你可以查看 `ResourceLoader` 文档来了解该枚举的值。
        // 调用默认的 `load()` 方法时，`cache_mode` 为 `CacheMode::REUSE`。
        cache_mode: i32,
    ) -> Variant {
        // TODO: 在此处添加保存逻辑，使用 `GFile` API（见下方链接）。

        // 如果加载操作失败并且你希望处理错误，
        // 可以返回 `godot::global::Error` 并将其转换为 `Variant`。
    }
}
```

链接到 `CacheMode` ([Godot][godot-cachemode], [Rust][api-cachemode]) 和 [`GFile`][api-gfile] 文档。

[godot-cachemode]: https://docs.godotengine.org/en/stable/classes/class_resourceformatloader.html#enum-resourceformatloader-cachemode
[api-cachemode]: https://godot-rust.github.io/docs/gdext/master/godot/classes/resource_loader/enum.CacheMode.html

[godot-saverflags]: https://docs.godotengine.org/en/stable/classes/class_resourcesaver.html#enum-resourcesaver-saverflags
[api-saverflags]: https://godot-rust.github.io/docs/gdext/master/godot/classes/resource_saver/struct.SaverFlags.html
[api-gfile]: https://godot-rust.github.io/docs/gdext/master/godot/prelude/struct.GFile.html

[godot-saver]: https://docs.godotengine.org/en/stable/classes/class_resourceformatsaver.html
[godot-loader]: https://docs.godotengine.org/en/stable/classes/class_resourceformatloader.html

[godot#101543]: https://github.com/godotengine/godot/pull/101543
[book#65]: https://github.com/godot-rust/book/pull/65#issuecomment-2585403123
