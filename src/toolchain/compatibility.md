<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 兼容性和稳定性

godot-rust 库同时支持多个 Godot 稳定版本。


## 与 Godot 的兼容性

在开发扩展库（或简称“扩展”）时，你需要考虑目标的引擎版本。
这里有两个概念上不同的版本：

- **API 版本**：你的扩展**编译**时所针对的 GDExtension 版本。
- **运行时版本**：指的是用 godot-rust 构建的库**运行**的 Godot 版本。

这两个版本可以不同，但需遵循以下规则：


### 当前保证

最新的 godot-rust 至少需要 **Godot 4.2**。

从该版本正式发布起，只要 _运行时版本 **>=** API 版本_ ，扩展可以在任何 Godot 版本中加载，。

- 你可以在 Godot `4.2.1` 或 `4.3` 中运行 `4.2` 的扩展。
- 你不能在 Godot `4.2.1` 中运行 `4.3` 的扩展。

只要 GDExtension API 以向后兼容的方式演进——自 Godot 4.1 以来它已显著实现了这一点——我们将尽力维持此保证。
如果你发现任何不一致之处，请向我们报告。


## 兼容性矩阵

我们通常会在 Godot 版本发布后提供 1-2 年的支持，具体取决于功能集和维护工作量。
例如，Godot 4.0 扩展与较新版本二进制不兼容，因此价值非常有限。
Godot 4.1 也缺乏 Rust 可调用对象、类型化信号、热重载等必要的基础功能。

如果你需要支持较旧的 Godot 版本，可以回退到旧的 godot-rust 版本。
但这些版本将不再接收任何更新，即使是关键错误修复。

| godot-rust 版本 | 最低 Godot 版本 | Godot 发布日期[^Godot-versions] |
|--------------------|-----------------------|-------------------------------------|
| 0.4+               | 4.2                   |  2023 11月                     |
| 0.2, 0.3           | 4.1                   |  2023 7月                          |
| 0.1                | 4.0[^Godot-4-0]       |  2023 3月                     |


### 开发理念

我们非常重视与引擎的兼容性，力求构建一个与多个 Godot 版本兼容的扩展生态系统。
没有什么比更新引擎后需要重新编译 10 个插件/扩展更让人头疼了。

但这有时会很困难，因为：

- Godot 可能会引入一些我们没有意识到的微小的破坏性变更。
- 一些在 C++ 和 GDScript 中不破坏兼容性的变更，在 Rust 中可能会导致破坏性变化（例如，给以前必需的参数提供默认值）。
- 使用新特性时，需要为旧版 Godot 提供回退或填充方案。

我们会在多个 Godot 版本上运行持续集成（CI）任务，以确保更新不会破坏兼容性。然而，可能的版本组合非常多，而且还在不断增加，所以我们可能会漏掉某些问题。
如果你发现不兼容或违反以下规则的情况，请告知我们。


### 不在支持范围内

我们**不会**投入精力维护以下内容的兼容性：

1. Godot 的开发版本，除非是最新的`master`分支。
   - 请注意，我们可能需要一些时间来跟进最新的变更，因此请不要在上游更改合并后的几天内报告问题。
2. 非稳定版本（alpha、beta、RC）。
3. 第三方绑定或 GDExtension API（如 C#、C++、Python 等）。
   - 这些可能有自己的版本保证和发布周期，也可能有特定的集成问题。如果你在 gdext 和其他绑定中发现问题，请先在 GDScript 中重现问题，以确保问题与我们相关。
   - 不过，我们会保持与 Godot 的兼容性，因此如果集成通过引擎进行（例如，Rust 调用一个 C# 实现的方法），应该是可以正常工作的。
4. 使用非标准构建标志的 Godot（例如，禁用某些模块）。
5. Godot分支或使用第三方模块的 Godot 引擎。


## Rust API的稳定性

godot-rust 的许多基础已经构建完成并处于生产就绪状态。然而，我们仍然定期添加新功能，有时会优化现有 API。
因此，**可以预期偶尔会有破坏性变化**。
这些变更通常是微小的，并会在[更新日志][changelog]和[迁移指南][migrate]中宣布。
我们还会在 API 中使用弃用机制，以实现平滑过渡。

需要注意的是，许多破坏性变化是由外部因素引起的，比如：

- GDExtension 发生了无法从用户角度抽象的变化。
- 类型系统或运行时保证中的一些细微差别，可以通过更好、更安全的方式被构建（例如类型化数组、RIDs）。
- 我们收到了来自游戏开发者和其他用户的反馈，认为某些工作流程非常繁琐。

我们在 [crates.io releases](https://crates.io/crates/godot)上的发布遵循 SemVer，但可能落后于 master 分支。

[changelog]: https://github.com/godot-rust/gdext/blob/master/Changelog.md
[migrate]: https://godot-rust.github.io/book/migrate

<br>

---

**脚注**

[^Godot-4-0]: Every extension developed with API version `4.0.x` **MUST** be run with the same runtime version.
    In particular, it is not possible to run an extension compiled with API version `4.0.x` in Godot 4.1 or later.
    This is due to breaking changes in Godot's GDExtension API.

[^Godot-versions]: See _Release history_ on [Wikipedia](https://en.wikipedia.org/wiki/Godot_(game_engine)#Release_history).
