<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 生态系统（Ecosystem）

本章列出了扩展 godot-rust 额外功能的第三方项目：工具、库、集成、应用等。
这些项目按类型和各自的领域进行分组（尽管这种分类并不总是完全明确）。

如果你想添加一个项目，请阅读[贡献指南](#贡献指南)！

另外，我们计划为游戏项目单独创建一个列表，并会在一个独立的页面上展示。


## 目录

<!-- toc -->

## 第三方项目列表


### 🏛️ Rust 库

| 项目                                                                        | 相关链接                                                            | 活跃度                                      |
|--------------------------------------------------------------------------------|--------------------------------------------------------------------------|-----------------------------------------------|
| 🌀 _**异步**_                                                                 |                                                                          |                                               |
| **[gdext-coroutines]**<br/>将 Rust 协程与 Godot 的 async/await 集成。 | [crates.io][gdext-coroutines-crate], [Discord][gdext-coroutines-discord] | ![gdext-coroutines][gdext-coroutines-badge]   |
| **[godot-tokio]**<br/>为 godot-rust 创建 Tokio 运行时。          | [crates.io][godot-tokio-crate], [Discord][godot-tokio-discord]           | ![godot-tokio][godot-tokio-badge]             |
| ___________________________________________________                            |                                                                          |                                               |
| 🏗️ _**项目工作流程**_                                                     |                                                                          |                                               |
| **[gd-rehearse]**<br/>为 godot-rust 代码编写单元测试。                      | [Discord][gd-rehearse-discord]                                           | ![gd-rehearse][gd-rehearse-badge]             |
| **[gd-props]**<br/>使用 `serde` 进行资源序列化。                       | [Discord][gd-props-discord]                                              | ![gd-props][gd-props-badge]                   |
| **[gdext-generation]**<br/>自动生成 `.gdextension` 文件。              | [Discord][gdext-generation-discord]                                      | ![gdext-generation][gdext-generation-badge]   |
| **[godot-rust-cli]**<br/>为 Godot 提供的 Rust CLI 脚本。                    | [Discord][godot-rust-cli-discord]                                        | ![godot-rust-cli][godot-rust-cli-badge]   |
| ___________________________________________________                            |                                                                          |                                               |
| 📜 _**脚本编程**_                                                             |                                                                          |                                               |
| **[godot-rust-script]**<br/>允许将 Rust 脚本添加到节点。         |                                                                          | ![godot-rust-script][godot-rust-script-badge] |
| ___________________________________________________                            |                                                                          |                                               |
| 🎮 _**游戏开发**_                                                      |                                                                          |                                               |
| **[SpireTween]**<br/>Godot 4.2+ 的替代tween库。                   | [Discord][spire-tween-discord]                                           | ![SpireTween][spire-tween-badge]              |
| **[GridForge]**<br/>网格地图的通用抽象。                         | [Discord][gridforge-discord]                                             | ![GridForge][gridforge-badge]                 |

[gdext-coroutines]: https://github.com/Houtamelo/gdext_coroutines
[gdext-coroutines-crate]: https://crates.io/crates/gdext_coroutines
[gdext-coroutines-discord]: https://discord.com/channels/723850269347283004/1255555232390451293/125555523
[gdext-coroutines-badge]: https://img.shields.io/github/last-commit/Houtamelo/gdext_coroutines

[godot-tokio]: https://github.com/2-3-5-41/godot_tokio
[godot-tokio-discord]: https://discord.com/channels/723850269347283004/1312490414762364928/1312490414762364928
[godot-tokio-crate]: https://crates.io/crates/godot_tokio
[godot-tokio-badge]: https://img.shields.io/github/last-commit/2-3-5-41/godot_tokio

[gd-rehearse]: https://github.com/StatisMike/gd-rehearse
[gd-rehearse-discord]: https://discord.com/channels/723850269347283004/1179891414474178661/1179891414474178661
[gd-rehearse-badge]: https://img.shields.io/github/last-commit/StatisMike/gd-rehearse

[gd-props]: https://github.com/StatisMike/gd-props
[gd-props-discord]: https://discord.com/channels/723850269347283004/1166451642145701989/1166451642145701989
[gd-props-badge]: https://img.shields.io/github/last-commit/StatisMike/gd-props

[gdext-generation]: https://github.com/sylbeth/gdext-generation
[gdext-generation-discord]: https://discord.com/channels/723850269347283004/1316664276819247124
[gdext-generation-badge]: https://img.shields.io/github/last-commit/sylbeth/gdext-generation

[godot-rust-cli]: https://github.com/TheColorRed/godot-rust
[godot-rust-cli-badge]: https://img.shields.io/github/last-commit/TheColorRed/godot-rust
[godot-rust-cli-discord]: https://discord.com/channels/723850269347283004/1325220721340977253

[godot-rust-script]: https://github.com/titannano/godot-rust-script
[godot-rust-script-badge]: https://img.shields.io/github/last-commit/titannano/godot-rust-script

[SpireTween]: https://github.com/Houtamelo/spire_tween
[spire-tween-discord]: https://discord.com/channels/723850269347283004/1257474308939452477/1257474308939452477
[spire-tween-badge]: https://img.shields.io/github/last-commit/Houtamelo/spire_tween

[GridForge]: https://github.com/StatisMike/grid-forge
[gridforge-discord]: https://discord.com/channels/723850269347283004/1238991002799444049/1238991002799444049
[gridforge-badge]: https://img.shields.io/github/last-commit/StatisMike/grid-forge


### 🧩 编辑器插件

| 项目                                                                       | 相关链接                           | 活跃度                                            |
|-------------------------------------------------------------------------------|-----------------------------------------|-----------------------------------------------------|
| 📐 _**用户界面**_                                                       |                                         |                                                     |
| **[Godot-Tour]**<br/>为编辑器和游戏内提供 UI 导览/教程。               | [Discord][godot-tour-discord]           | ![Godot-Tour][godot-tour-badge]                     |
| ___________________________________________________                           |                                         |                                                     |
| 🎨 _**图形**_                                                             |                                         |                                                     |
| **[Godot Trail 3D]**<br/>为 Godot 添加 `Trail3D` 节点。                      | [Discord][godot-trail-3d-discord]       | ![Godot Trail 3D][godot-trail-3d-badge]             |
| ___________________________________________________                           |                                         |                                                     |
| 🧲 _**物理**_                                                              |                                         |                                                     |
| **[Godot Rapier Physics]**<br/>为 Godot 提供 Rapier 2D 和 3D 物理引擎集成。    | [Discord][godot-rapier-physics-discord] | ![Godot Rapier Physics][godot-rapier-physics-badge] |
| **[Godot Rapier 3D]**<br/>启用 Godot 使用 Rapier 物理引擎的 GDExtension。 | [Discord][godot-rapier-3d-discord]      | ![Godot Rapier 3D][godot-rapier-3d-badge]           |
| ___________________________________________________                           |                                         |                                                     |
| 🧙‍♂️ _**叙事**_                                                      |                                         |                                                     |
 | **[nobodywho]**<br/>与本地 LLM 互动进行互动式故事讲述。    | [Discord][nobodywho-discord]            | ![nobodywho][nobodywho-badge]                       |
| ___________________________________________________                           |                                         |                                                     |
| 🏗️ _**项目工作流程**_                                                    |                                         |                                                     |
| **[godot-sandbox]**<br/>为 C++、Rust 和其他语言提供安全的mod支持。      |                                         | ![godot-sandbox][godot-sandbox-badge]               |
| ___________________________________________________                           |                                         |                                                     |
| 🌐 _**本地化**_                                                        |                                         |                                                     |
| **[Fluent Translation]**<br/>使用 Mozilla 的 Fluent (FTL) 进行翻译。       | [Asset Library][godot-fluent-translation-assetlib] | ![godot-fluent-translation][godot-fluent-translation-badge] |

[Godot-Tour]: https://github.com/Decapitated/Godot-Tour
[godot-tour-discord]: https://discord.com/channels/723850269347283004/1272688558070698037/1272688558070698037
[godot-tour-badge]: https://img.shields.io/github/last-commit/Decapitated/Godot-Tour

[Godot Trail 3D]: https://github.com/SomeRanDev/Godot-Trail3D
[godot-trail-3d-discord]: https://discord.com/channels/723850269347283004/1246199893043974247/1246199893043974247
[godot-trail-3d-badge]: https://img.shields.io/github/last-commit/SomeRanDev/Godot-Trail3D

[Godot Rapier 3D]: https://github.com/deltasiege/godot-rapier-3d
[godot-rapier-3d-discord]: https://discord.com/channels/723850269347283004/1238758369767198741/1238758369767198741
[godot-rapier-3d-badge]: https://img.shields.io/github/last-commit/deltasiege/godot-rapier-3d

[Godot Rapier Physics]: https://github.com/appsinacup/godot-rapier-physics
[godot-rapier-physics-discord]: https://discord.com/channels/723850269347283004/1233345975255433266/1233345975255433266
[godot-rapier-physics-badge]: https://img.shields.io/github/last-commit/appsinacup/godot-rapier-physics

[nobodywho]: https://github.com/nobodywho-ooo/nobodywho
[nobodywho-discord]: https://discord.com/channels/723850269347283004/1309111775991693332/1309111775991693332
[nobodywho-badge]: https://img.shields.io/github/last-commit/nobodywho-ooo/nobodywho

[godot-sandbox]: https://github.com/libriscv/godot-sandbox
[godot-sandbox-badge]: https://img.shields.io/github/last-commit/libriscv/godot-sandbox

[Fluent Translation]: https://github.com/RedMser/godot-fluent-translation
[godot-fluent-translation-assetlib]: https://godotengine.org/asset-library/asset/2937
[godot-fluent-translation-badge]: https://img.shields.io/github/last-commit/RedMser/godot-fluent-translation


### 🖥️ 应用

| 项目                                                                 | 相关链接                          | 活跃度                                          |
|-------------------------------------------------------------------------|----------------------------------------|---------------------------------------------------|
| 🎛️ _**软件平台**_                                            |                                        |                                                   |
| **[Godot Boy]**<br/>用 Rust 编写的 Game Boy 模拟器。       | [Discord][godot-boy-discord]           | ![Godot Boy][godot-boy-badge]                     |
| **[GDScript Transpiler]**<br/>用 Rust 重新实现部分 GDScript 功能。   | [Discord][gdscript-transpiler-discord] | ![GDScript Transpiler][gdscript-transpiler-badge] |
| ___________________________________________________                     |                                        |                                                   |
| 🛸 _**技术演示**_                                                     |                                        |                                                   |
| **[Godot boids]**<br/>为 Godot 添加 2D/3D 集群运动（flocking）的插件。 | [Discord][godot-boids-discord]         | ???                                               |

[Godot Boy]: https://gitlab.com/greenfox/godot-boy
[godot-boy-discord]: https://discord.com/channels/723850269347283004/1230789480290586624/1230789480290586624
[godot-boy-badge]: https://img.shields.io/gitlab/last-commit/greenfox/godot-boy

[GDScript Transpiler]: https://gitlab.com/the-SSD/gdscript-transpiler
[gdscript-transpiler-badge]: https://img.shields.io/gitlab/last-commit/the-SSD/gdscript-transpiler
[gdscript-transpiler-discord]: https://discord.com/channels/723850269347283004/1237464552384499833/1237464552384499833

[Godot boids]: https://git.gaze.systems/dusk/godot_boids
[godot-boids-discord]: https://discord.com/channels/723850269347283004/1279645654439821393/1279645654439821393


## 贡献指南

如果你有一个适合添加到这个列表的项目，太好了！你不需要是作者——如果你发现了能让其他人受益的东西，请分享出来！

为了保持这个列表对访问者有用，以下是一些接受标准：

- 项目必须与 godot-rust 相关（不仅是 Rust 或仅是 Godot）。应使用 Godot 4。
- 项目已有一定的实质内容，至少有最小的文档/示例。
  - 这可以是一个在 GitHub 上可用的库，一个有效的演示等。无需发布 crate 或非常精美的展示；关键是项目对新手来说是可以访问的。
  - 如果你想讨论想法和正在进行的原型，欢迎 [在Discord 的 `#showcase`][discord-showcase] 频道开启讨论！
- 作者应愿意维护该项目一段时间。
  - GDExtension 在二进制兼容性方面表现非常好，[godot-rust 支持的扩展可以向下兼容到 Godot 4.1][gdext-compat]。
    所以如果你通过扩展（例如作为编辑器插件）集成，你的项目通常会比源代码更具未来兼容性。
  - 话虽如此，我们通常不会经常做重大破坏性更改。
- 如果该项目打算分发和使用，请确保它附带了许可证（例如软件的开源许可证，或艺术作品的 Creative Commons 许可证）。

完成这些步骤后，请直接向 [文档repo][book-repo] 提交一个 pull request。如果你不确定是否符合标准或有其他问题，随时可以在 Discord 或 [文档issue追踪器][book-issues] 提问。

```admonish tip title="一个蓬勃发展的生态系统"
每一个项目都在丰富 Godot 和 Rust 生态系统，让更多的人享受游戏开发的乐趣。
非常感谢每一位贡献者！
```

[discord-showcase]: https://discord.com/channels/723850269347283004/1163944783484563537
[gdext-compat]: ../toolchain/compatibility.md
[book-repo]: https://github.com/godot-rust/book
[book-issues]: https://github.com/godot-rust/book/issues
