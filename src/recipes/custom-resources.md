<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 自定义 resources

自定义 `Resource` 类型可以暴露给终端用户，在他们的开发过程中使用。`Resource` 类型能够存储可以在编辑器 GUI 中轻松编辑的数据。例如，您可以创建一个自定义的 `AudioStream` 类型，用来处理一种新颖且有趣的音频文件类型。


## 注册 `Resource`

这个工作流与  [Hello World example][hello]类似：

```rust
#[derive(GodotClass)]
#[class(init, base=Resource)]
struct ResourceType {
    base: Base<Resource>,
}
```

上述resource没有导出任何变量。虽然并非所有resource都需要导出变量，但大多数resource都需要。

如果你的自定义resource有需要在编辑器中运行的生命周期方法（例如 `ready()`、`process()` 等），你应该使用 `#[class(tool)]` 注解该类。

```rust
#[derive(GodotClass)]
#[class(tool, init, base=Resource)]
struct ResourceType {
    base: Base<Resource>,
}

#[godot_api]
impl IResource for ResourceType {
    // String representation of the object.
    fn to_string(&self) -> GString { ... }

    // Called when the object receives a Godot notification.
    fn on_notification(&mut self, what: ObjectNotification) { ... }

    // Called whenever get() is called or Godot gets the value of a property.
    fn get_property(&self, property: StringName) -> Option<Variant> { ... }

    // Called whenever Godot set() is called or Godot sets the value of a property.
    fn set_property(&mut self, property: StringName, value: Variant) -> bool { ... }

    // Called whenever Godot get_property_list() is called,
    // the returned vector here is appended to the existing list of properties.
    fn get_property_list(&mut self) -> Vec<PropertyInfo> { ... }

    // Called whenever Godot retrieves value of property. Allows to customize existing properties.
    // Every property info goes through this method, except properties added with get_property_list().
    fn validate_property(&self, property: &mut PropertyInfo) { ... }

    // Called by Godot to tell if a property has a custom revert or not.
    fn property_get_revert(&self, property: StringName) -> Option<Variant> { ... }

    fn setup_local_to_scene(&mut self) { ... }
}
```

与在 GDScript 中定义自定义resources类似，重要的是将这个类标记为“工具类”，这样它才可以在编辑器中使用。

关于如何注册函数、属性等系统，可以在 [注册 Rust 符号][register] 部分找到详细描述。


[hello]: ../intro/hello-world.md
[register]: ../register/index.html
