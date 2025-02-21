<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# Hello World

本页面将向您展示如何开发自己的小型扩展库并从 Godot 加载它。
本教程深受官方 Godot 文档中 [创建第一个脚本][tutorial-begin] 的启发。
如果您对某些 GDScript 概念如何映射到 Rust 感兴趣，我们建议您跟随该教程。


## 目录
<!-- toc -->


## 目录结构设置

我们假设项目使用以下的文件结构，其中 Godot 和 Rust 存放在不同的文件夹：

```txt
📂 project_dir
│
├── 📂 .git
│
├── 📂 godot
│   ├── 📂 .godot
│   ├── 📄 HelloWorld.gdextension
│   └── 📄 project.godot
│
└── 📂 rust
    ├── 📄 Cargo.toml
    ├── 📂 src
    │   └── 📄 lib.rs
    └── 📂 target
        └── 📂 debug
```


## 创建 Godot 项目

要使用 godot-rust，您需要安装 4.1 或更高版本的 Godot。您可以随时下载最新稳定版。您也可以下载开发中的版本，
但我们对开发中的版本 [不提供官方支持][compatibility]，因此推荐使用稳定版。

打开 Godot 项目管理器，在 `godot/` 子文件夹中创建一个新的 Godot 4 项目，并向新场景(scene)的中心添加一个 `Sprite2D`。
我们建议您跟随 [官方教程][tutorial-begin]，并在它要求您创建脚本时停下。

运行您的场景以确保一切正常。保存更改，并考虑使用 Git 版本控制来管理本教程中的每一步。


## 创建Rust crate

要使用 Cargo 创建一个新的 crate，打开终端，导航到目标文件夹，然后输入：

```bash
cargo new "{YourCrate}" --lib
```

其中 {YourCrate} 将作为您选择的 crate 名称的占位符。为了与文件结构保持一致，我们选择 Rust 作为 crate 名称。
使用 --lib 创建一个库（而非可执行文件），但是这个 crate 还需要一些额外的配置。

其中 `{YourCrate}` 将作为您选择的 crate 名称的占位符。为了与文件结构保持一致，我们选择 `rust` 作为 crate 名称。使用 `--lib` 创建一个库（而非可执行文件），但是这个 crate 还需要一些额外的配置。

打开`Cargo.toml`文件并按以下方式修改：

```toml
[package]
name = "rust_project" # 动态库名称的一部分; 我们使用 {YourCrate} 作为占位符
version = "0.1.0"     # 你目前可以保持版本和版次不变
edition = "2021"

[lib]
crate-type = ["cdylib"]  # 将此crate编译为动态C库 （dynamic C library）.
```

`cdylib`是 Rust 中不常见的 crate 类型。与构建应用程序（`bin`）或供其他 Rust 代码使用的库（`lib`）不同，我们创建了一个 _动态_ 库，暴露 C 语言接口。
这个动态库将在运行时通过 GDExtension 接口加载到 Godot 中。

现在，使用以下命令将 gdext 添加到您的项目中：


```bash
cargo add godot
```

每次编写代码时，您可以像其他 Rust 项目一样使用 `cargo` 进行编译：

```bash
cargo build
```

根据您的设置，这应该至少输出一个编译后的库变体到  `{YourCrate}/target/debug/`目录


```admonish tip
如果您希望跟进最新的开发（并承担相关风险），您可以直接在 `Cargo.toml` 的 `[dependencies]` 部分链接到 GitHub 仓库。  
为此，请将：
~~~toml
godot = "0.x.y"
~~~
替换为：
~~~toml
godot = { git = "https://github.com/godot-rust/gdext", branch = "master" }
~~~

```


## 将 Godot 与 Rust 连接


### `.gdextension`  文件

此文件告诉 Godot 如何加载您的编译后的 Rust 扩展。它包含动态库的路径以及初始化它的入口点（函数）。

首先，在 `godot` 子文件夹中的任何位置添加一个空的 `.gdextension` 文件。如果您熟悉 Godot 3，它相当于 `.gdnlib`。
在本例中，我们在 `godot` 子文件夹中创建了 `res://HelloWorld.gdextension`，并按以下方式填充：


```ini
[configuration]
entry_symbol = "gdext_rust_init"
compatibility_minimum = 4.1
reloadable = true

[libraries]
linux.debug.x86_64 =     "res://../rust/target/debug/lib{YourCrate}.so"
linux.release.x86_64 =   "res://../rust/target/release/lib{YourCrate}.so"
windows.debug.x86_64 =   "res://../rust/target/debug/{YourCrate}.dll"
windows.release.x86_64 = "res://../rust/target/release/{YourCrate}.dll"
macos.debug =            "res://../rust/target/debug/lib{YourCrate}.dylib"
macos.release =          "res://../rust/target/release/lib{YourCrate}.dylib"
macos.debug.arm64 =      "res://../rust/target/debug/lib{YourCrate}.dylib"
macos.release.arm64 =    "res://../rust/target/release/lib{YourCrate}.dylib"
```

`[configuration]`部分应照原样复制。

- `entry_symbol`是指 gdext 暴露的入口点函数。我们选择 `"gdext_rust_init"`，这是 **gdext** 的默认值（但如果需要，可以配置）。
- `compatibility_minimum` 指定了扩展所需的最低 **Godot** 版本。使用低于该版本的 **Godot** 打开项目将导致扩展无法运行。
  - 如果您要构建一个供他人使用的插件，请尽量将此版本设置得尽可能低，以实现更广泛的生态系统兼容性，但这可能会限制您使用的功能。
- `reloadable` 指定当编辑器窗口失去焦点后再恢复时，应重新加载扩展。有关更多详情，请参阅 [Godot issue #80284][gdextension-reloadable]。
  - 如果 Godot 崩溃，您可能需要尝试关闭或移除此设置。

`[libraries]` 部分应根据您的动态 Rust 库的路径进行更新。

- 左侧的键是 **Godot** 项目的构建目标平台。
  - 请参考 [GDExtension 文档][godot-build-targets] 以了解更多可能的值。
- 右侧的值是你的动态库的文件路径。
  - `res://` 前缀表示文件路径是相对于 **Godot** 目录 的，无论您的 `HelloWorld.gdextension` 文件位于何处。您可以在 [Godot 资源路径][godot-resource-paths] 中了解更多。
  - 如果您记得文件结构，`godot` 和 `rust` 文件夹是兄弟关系，因此我们需要回到上一级目录才能访问 `rust`。
- 如果您计划将项目导出到其他平台，您可以为多个平台添加配置。
至少，您需要为当前操作系统的 `debug` 模式配置路径。

```admonish tip
您还可以使用符号链接和 git 子模块，然后将它们当作普通文件夹和文件来处理。Godot 也能正常读取它们！
```

```admonish note title="导出路径"
导出项目时，您需要使用 `res://` _内部_ 的路径。  
不支持像 `..` 这样的外部路径。
```

```admonish note title="自定义 Rust 目标"
如果您通过 `--target` 标志或 `.cargo/config.toml` 文件指定了 Cargo 编译目标，Rust 库将被放置在包含目标架构的路径下，
而 `.gdextension` 文件中的库路径需要匹配。例如，对于 M1 Mac（`macos.debug.arm64` 和 `macos.release.arm64`），路径应为  
`"res://../rust/target/aarch64-apple-darwin/debug/lib{YourCrate}.dylib"`。

```


### `extension_list.cfg`

在您第一次打开 Godot 编辑器时，会自动生成一个名为 `res://.godot/extension_list.cfg` 的文件。
此文件列出了项目中注册的所有扩展。如果该文件不存在，您也可以手动创建它，仅包含到你的`.gdextension` 文件的 Godot 路径：

```text
res://HelloWorld.gdextension
```


## 您的第一个 Rust 扩展

```admonish note title=".gdignore"
如果您没有遵循 [推荐的 gdext 项目目录结构设置][directory-setup]，将 `rust/` 和 `godot/` 目录分开，  
而是将 Rust 源代码直接放入 Godot 项目中，那么请考虑在 Rust 代码根目录添加 [.gdignore][gd-ignore] 文件。
这可以避免 Rust 编译器在 Rust 文件夹中生成扩展名模糊的文件（如 `.obj`），而 Godot 编辑器可能错误地尝试导入它们，从而导致错误并阻止您构建项目。

```


### Rust入口点

如前所述，我们编译的 C 库需要暴露一个 _入口点_ 给 Godot：一个可以通过 GDExtension 调用的 C 函数。
设置此项需要一些底层的 [FFI][wikipedia-ffi] 代码，gdext 为您抽象了这些细节。

在你的 `lib.rs` 文件中，将模板替换为以下内容：


```rust
use godot::prelude::*;

struct MyExtension;

#[gdextension]
unsafe impl ExtensionLibrary for MyExtension {}
```

这里有几个要点：

1. 将[`prelude`][api-prelude]模块从 [`godot`][api-godot] crate 引入作用域。
    该模块包含了 gdext API 中最常用的符号。
2. 定义一个名为 `MyExtension` 的结构体。它只是一个类型标记，没有数据或方法，您可以根据需要命名它。
3. 为该类型实现 [`ExtensionLibrary`][api-extensionlibrary] trait，并用 `#[gdextension]` 属性标记。

最后这一点声明了实际的 GDExtension 入口点，proc-macro 属性会处理底层的细节。


### 故障排除


首次设置时常会遇到一些问题。特别是与库无法找到或 `gdext_rust_init` 入口点符号缺失或无法解析相关的错误，通常是由于初始设置不正确。
以下是一些故障排除步骤，应该能解决大部分常见问题。


- 您是否运行了 `cargo build`?
- 在 `Cargo.toml`, ，是否设置了 `crate-type = ["cdylib"]`?
- 在  `my-extension.gdextension`中，是否设置了 `entry_symbol = "gdext_rust_init"`? 没有其他符号可以正常运行。
- `my-extension.gdextension`  中的路径设置是否正确？
  - 您确定吗？请仔细检查 `/rust/target/debug/`目录，确保`.so`/`.dll`/`.dylib` 文件的名称是否拼写正确。
  - 路径也必须相对于  `project.godot` 所在的目录。通常情况下，应该是`res://../rust/...`。
- 您是否编写了生成入口点符号所需的 Rust 代码？
  - 请参阅上面的Rust入口点 部分了解如何操作
- 您的 gdext 和 Godot 版本是否兼容？请查看 [此页面][versioning] 以了解如何选择正确的版本。
- 如果您使用 `api-custom`，请确认您是否：
- 将 Godot 设置在您的 `PATH` 中为 `godot4`,
- 或者设置了名为 `GODOT4_BIN`，包含 Godot 可执行文件的路径？
- 您的目录结构是否如下所示？如果是这样，寻求帮助时会更容易。

```txt
my-cool-project
├── godot
│   ├── project.godot
│   └── my-extension.gdextension
└── rust
    ├── Cargo.toml
    ├── src
    └── target
        └── debug
            └── (lib)?my_extension.(so|dll|dylib)
```


## 创建一个 Rust 类

现在，让我们编写 Rust 代码来定义一个可以在 Godot 中使用的  _类_。

每个类都继承一个现有的 Godot 提供的类（它的 _基类_ 或简称 _base_）。
Rust 本身不支持继承，但 gdext API 在某种程度上模拟了它。


### 类的声明

在本例中，我们声明一个名为 `Player` 的类，它继承自 `Sprite2D`（一个node类型）。
这可以在 `lib.rs` 中定义，也可以在单独的 `player.rs` 文件中定义。
如果选择后者，请不要忘记在 `lib.rs` 文件中声明 `mod player`;。

```rust
use godot::prelude::*;
use godot::classes::Sprite2D;

#[derive(GodotClass)]
#[class(base=Sprite2D)]
struct Player {
    speed: f64,
    angular_speed: f64,

    base: Base<Sprite2D>
}
```

我们来逐步解释。

1. `godot`prelude包含了最常用的符号。较少使用的类位于 [engine][api-class-engine] 模块中。
2. `#[derive]` 属性将 `Player` 注册为 Godot 引擎中的类。
详细信息请参考 [API 文档][api-derive-godotclass] 中关于 `#[derive(GodotClass)]` 的说明。
3. 可选的 `#[class]`  属性配置类的注册方式。在本例中，我们指定 `Player` 继承 Godot 的 `Sprite2D` 类。
如果不指定 `base` 键，则基类将隐式为 `RefCounted`，就像在 GDScript 中省略 `extends`关键字一样。
4. 我们为逻辑定义了两个字段 `speed` 和 `angular_speed`。这些是普通的 Rust 字段，没有特别的地方。稍后会介绍它们的用途。
5. `Base<T>` 类型用于 `base` 字段，它允许通过组合访问基类实例（因为 Rust 不支持继承）。这使得可以通过扩展 trait 访问两个方法 `self.base()` 和 `self.base_mut()`
   - `T` 必须与声明的基类匹配。例如， `#[class(base=Sprite2D)]` 与 `Base<Sprite2D>`.
   - 名称可以自由选择，但 `base` 是常见的习惯。
   - 你 _可以不_ 声明此字段。如果缺少此字段，则无法在 `self` 内部访问基类对象。
      例如，继承自 `RefCounted` 的数据包通常不需要此字段。

```admonish warning title="正确的 node 类型"
将 `Player` 类实例添加到场景时，请确保选择节点类型为 `Player` **而不是它的基类 `Sprite2D`**。  
否则，您的 Rust 逻辑将无法运行。稍后当您准备好进行测试时，我们将指导您进行更改你的场景。

如果 Godot 无法加载 Rust 类（例如，由于扩展中的错误），它可能会默默地将其替换为基类。
使用版本控制（git）检查 .tscn 文件中是否有你不想要的更改发生。
```


### 方法声明

现在，让我们添加一些逻辑。我们首先重写 `init` 方法，也就是构造函数。
这对应于 GDScript 的 `_init()` 函数。


```rust
use godot::classes::ISprite2D;

#[godot_api]
impl ISprite2D for Player {
    fn init(base: Base<Sprite2D>) -> Self {
        godot_print!("Hello, world!"); // 输出到 Godot 控制台
        
        Self {
            speed: 400.0,
            angular_speed: std::f64::consts::PI,
            base,
        }
    }
}
```

同样，我们逐一说明这里协同工作的部分：

1. `#[godot_api]` - 这告知 gdext 接下来的`impl`块是 Rust API，供 Godot 使用。
    这里是必需的；忘记添加会导致编译错误。
2. `impl ISprite2D` - 每个引擎类都有一个 `I{ClassName}` trait，包含该类的虚函数以及一般用途的功能，例如 `init`（构造函数）或 `to_string`（字符串转换）。
    此 trait 没有必需的方法。
3. `init` 构造函数是一个关联函数（其他语言中的“静态方法”），它以基类实例为参数并返回构造好的`Self`实例。
通常，基类实例只是传递给构造函数，构造函数是初始化其他字段的地方。在此示例中，我们为 `speed` 和 `angular_speed` 字段赋予初始值 `400.0` 和 `PI`。

现在初始化完成后，我们可以继续添加实际的逻辑。我们希望持续旋转sprite，因此重写 `process()` 方法。
这对应于 GDScript 的 `_process()`。如果您需要固定的帧率，请使用 `physics_process()`。

```rust
use godot::classes::ISprite2D;

#[godot_api]
impl ISprite2D for Player {
    fn init(base: Base<Sprite2D>) -> Self { /* 如前所述 */ }

    fn physics_process(&mut self, delta: f64) {
        // 在 GDScript中，这将是： 
        // rotation += angular_speed * delta
        
        let radians = (self.angular_speed * delta) as f32;
        self.base_mut().rotate(radians);
        // 'rotate' 方法需要一个 f32， 
        // 因此我们将 'self.angular_speed * delta' 的f64 转换为 f32
    }
}
```

GDScript 使用属性语法；而 Rust 需要显式的方法调用。另外，访问基类方法 —— 例如本例中的 `rotate()`，
需要通过 `base()` 和 `base_mut()` 方法来实现。

```admonish warning title="直接访问字段"
不要直接使用 `self.base` 字段。应使用 `self.base()` 或 `self.base_mut()`，否则您将无法访问并调用基类方法。
```

在这一点上，您应该可以看到结果。编译代码并启动 Godot 编辑器。
右键单击场景树中的 `Sprite2D`，选择 “更改类型”
在弹出的 “更改类型” 对话框中找到并选择 `Player` 节点类型，它将作为 `Sprite2D` 的子节点出现。

现在保存更改，并运行场景。sprite应当以恒定的速度旋转。

![rotating sprite][img-sprite-rotating]

```admonish tip
**启动 Godot 应用程序**


不幸的是，在 Godot 4.2 之前，存在 [GDExtension 限制][issue-no-reload]，该限制阻止在编辑器打开时重新编译。  
自 Godot 4.2 起，已支持热重载扩展。这意味着您可以重新编译 Rust 代码，  
Godot 会自动更新变更，而无需重新启动编辑器。

但是，如果您不需要修改编辑器本身，您可以从命令行或您的 IDE 启动 Godot。  
请查看 [命令行教程][godot-command-line] 了解更多信息。

```

我们现在将为sprite添加一个translation组件，参照 [Godot教程][tutorial-full-script]。

```rust
use godot::classes::ISprite2D;

#[godot_api]
impl ISprite2D for Player {
    fn init(base: Base<Sprite2D>) -> Self { /* as before */ }

    fn physics_process(&mut self, delta: f64) {
        // GDScript 代码：
        //
        // rotation += angular_speed * delta
        // var velocity = Vector2.UP.rotated(rotation) * speed
        // position += velocity * delta
        
        let radians = (self.angular_speed * delta) as f32;
        self.base_mut().rotate(radians);

        let rotation = self.base().get_rotation();
        let velocity = Vector2::UP.rotated(rotation) * self.speed as f32;
        self.base_mut().translate(velocity * delta as f32);
        
        // 或更详细的写法： 
        // let this = self.base_mut();
        // this.set_position(
        //     this.position() + velocity * delta as f32
        // );
    }
}
```

结果应该是一个带有偏移的旋转sprite。

![rotating translated sprite][img-sprite-moving]


### 自定义Rust APIs

假设您想为 `Player` 类添加一些可以从 GDScript 调用的功能。为此，您需要一个单独的 `impl` 块，同样标注 `#[godot_api]`。
然而，这次我们使用的是 _固有的_ impl（即没有 trait 名称）。

具体来说，我们添加一个函数来增加速度，并添加一个信号,当速度发生变化时通知其他对象。

```rust
#[godot_api]
impl Player {
    #[func]
    fn increase_speed(&mut self, amount: f64) {
        self.speed += amount;
        self.base_mut().emit_signal("speed_increased", &[]);
    }

    #[signal]
    fn speed_increased();
}
```

`#[godot_api]`再次起到将 API 暴露给 Godot 引擎的作用。但这里有两个新属性：

- `#[func]` 将函数暴露给 Godot。参数和返回类型会映射到对应的 GDScript 类型。

- `#[signal]` 声明一个信号。信号可以通过 `emit_signal` 方法触发（每个 Godot 类都提供了这个方法，因为它继承自 `Object`）。


API 属性通常遵循 GDScript 关键字的命名：`class`, `func`, `signal`, `export`, `var`等。

这就是 _Hello World_ 教程的全部内容！接下来的章节将更详细地介绍 gdext 提供的各种功能。


[api-class-engine]: https://godot-rust.github.io/docs/gdext/master/godot/classes/index.html
[api-class-sprite2d]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Sprite2D.html
[api-derive-godotclass]: https://godot-rust.github.io/docs/gdext/master/godot/register/derive.GodotClass.html
[api-extensionlibrary]: https://godot-rust.github.io/docs/gdext/master/godot/prelude/trait.ExtensionLibrary.html
[api-godot]: https://godot-rust.github.io/docs/gdext/master/godot/index.html
[api-prelude]: https://godot-rust.github.io/docs/gdext/master/godot/prelude/index.html
[compatibility]: ../toolchain/compatibility.md
[directory-setup]: https://godot-rust.github.io/book/intro/hello-world.html#directory-setup
[gd-ignore]: https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html#ignoring-specific-folders
[gdextension-reloadable]: https://github.com/godotengine/godot/pull/80284
[godot-build-targets]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/gdextension_cpp_example.html#using-the-gdextension-module
[godot-command-line]: https://docs.godotengine.org/en/stable/tutorials/editor/command_line_tutorial.html
[godot-resource-paths]: https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html#external-vs-built-in
[img-sprite-moving]: https://docs.godotengine.org/en/stable/_images/scripting_first_script_rotating_godot.gif
[img-sprite-rotating]: https://docs.godotengine.org/en/stable/_images/scripting_first_script_godot_turning_in_place.gif
[issue-no-reload]: https://github.com/godotengine/godot/issues/66231
[tutorial-begin]: https://docs.godotengine.org/en/stable/getting_started/step_by_step/scripting_first_script.html
[tutorial-full-script]: https://docs.godotengine.org/en/stable/getting_started/step_by_step/scripting_first_script.html#complete-script
[versioning]: https://godot-rust.github.io/book/toolchain/godot-version.html
[wikipedia-ffi]: https://en.wikipedia.org/wiki/Foreign_function_interface
