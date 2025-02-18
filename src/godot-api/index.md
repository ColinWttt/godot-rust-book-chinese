<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 使用 Godot API


在这一章中，您将学习如何使用 Rust 代码与 Godot 引擎进行交互。我们将首先介绍内置类型和对象，然后深入探讨引擎 API 调用，并讨论与之相关的 gdext 特定惯例。

如果您有兴趣将自己的 Rust 符号暴露给引擎和 GDScript 代码，请查阅[注册 Rust 符号](../register/index.md)这一章。不过强烈建议您先阅读本章，因为它介绍了重要的概念。

