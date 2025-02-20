<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 自定义节点图标

默认情况下，所有自定义类型在编辑器 UI 中都会使用 `Node` 图标——例如，在场景树中或选择创建节点时。

虽然这种方式可以使用，但您可能希望为节点类型添加自定义图标，特别是如果您计划将扩展分发给其他人。

所有图标必须通过其类名在 `.gdextension` 文件中进行注册。为此，您可以添加一个新的 `icon` 部分。类名是键，SVG 文件的路径是值。


```toml
[icons]

MyClass = "res://addons/your_extension/filename.svg"
```

```admonish note title="图标路径"
路径基于 `res://` 方案，与其他 Godot 资源类似。建议使用 Godot 的惯例，即创建一个 `addons` 文件夹，并在其中放置插件的名称。

更多关于此的解释，请参见 Godot 文档：

- [安装插件][godot-installing-plugins]

- [制作插件][godot-making-plugins]

```

[godot-installing-plugins]: https://docs.godotengine.org/en/stable/tutorials/plugins/editor/installing_plugins.html#finding-plugins
[godot-making-plugins]: https://docs.godotengine.org/en/stable/tutorials/plugins/editor/making_plugins.html


## 自定义图标的格式


Godot 文档中有一页专门介绍了创建自定义图标的工具和资源。概括来说：

- 使用 SVG 格式。
- 宽高比为正方形，16x16 单位是参考尺寸。
- 参考 [Godot 图标颜色映射][gh-godot-colors]。
  - 使用浅色模式的颜色——Godot 仅支持从浅色到深色的颜色转换，不支持深色到浅色的转换。


```admonish help "第三方文章"
用户 _QueenOfSquiggles_ 在她的个人博客上写了这篇文章的替代版本 [on her personal blog][qos-colors]，其中包括浅色和深色主题颜色的预览。

关于如何使用她的参考页面的详细信息，请见 [here][qos-info]。

```

[godot-icons]: https://docs.godotengine.org/en/stable/contributing/development/editor/creating_icons.html
[gh-godot-colors]: https://github.com/godotengine/godot/blob/master/editor/themes/editor_color_map.cpp
[qos-colors]: https://queenofsquiggles.github.io/tech/godot-icon-colours/
[qos-info]: https://queenofsquiggles.github.io/tech/godot-icon-colours/#how-to-use-this
