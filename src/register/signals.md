<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 注册信号  

信号是Godot中实现观察者模式的一种机制。你可以发射事件，这些事件会被所有订阅（“连接”）该信号的对象接收，从而解耦发送者和接收者。如果你之前没有使用过Godot的信号，强烈建议先阅读[GDScript教程][Godot-GDScript-signals]。


## 目录

<!-- toc -->

## GDScript信号的问题  

在GDScript中，你可以如下定义信号，参数名称和类型是可选的：  

```java
signal damage_taken  
signal damage_taken(amount)  
signal damage_taken(amount: int)  
```  

然而，上述声明之间的区别仅是信息性的（例如显示在类的文档中）。
来看一个例子：  

```java
signal damage_taken(amount: int)  

func log_damage():  
    print("damaged!")  

func _ready():  
    damage_taken.connect(log_damage)  
```  

注意`log_damage()`没有参数，但你连接它时既不会在解析时也不会在运行时收到警告。  

不止`connect`有这个问题，再看一个向`emit()`传递错误类型参数的例子：  

```php
signal damage_taken(amount: int)  

func log_damage(amount): # 现在带参数  
    print("damaged: ", amount)  

func _ready():  
    damage_taken.connect(log_damage)  
    damage_taken.emit(true) # 不是int，不用担心 -> print "damaged: true"  
```  

GDScript会愉快地传递`bool`，尽管信号声明为`int`。  

```admonish danger title="GDScript信号不是类型安全的"
在GDScript中，信号参数列表**不会进行类型检查**。  

不匹配的`connect()`或`emit()`调用可能会也可能不会在运行时被捕获，具体取决于handler函数自身的类型声明。
它们永远不会在解析时被捕获。  
```

虽然在上面的例子中这似乎是小问题，但在具有许多类似信号的大型项目中，尤其是在重构时，这会变得难以追踪。信号的设计目的是作为发送者和接收者之间的API接口——但除了高度的手动规范和测试外，无法验证这一接口契约。  


## Rust信号  

`godot-rust`提供了类型安全且直观的API来连接和发射信号，尽管它们在GDScript中没有类型。
你可以依赖签名，无需担心重构，因为Rust会在编译时捕获所有不匹配。  

在Rust中，信号可以通过`#[signal]`属性在`#[godot_api]`块中定义。
以之前的类为例，声明一个`damage_taken`信号：  

```rust
#[derive(GodotClass)]  
#[class(init, base=Node3D)]  
struct Monster {  
    hitpoints: i32,  
    base: Base<Node3D>, // 声明信号时需要。  
}  

#[godot_api]  
impl Monster {  
    #[signal]  
    fn damage_taken(amount: i32);  
}  
```  

信号的语法类似于`#[func]`，但需要用分号代替函数体。不支持接收者（`&self`、`&mut self`）和返回类型。  

---


### 生成的代码  

一旦注册了至少一个信号，`godot-rust`会为你的类实现[`WithUserSignals`][api-withusersignals] trait。
这提供了`signals()`方法，现在可以在类方法中访问。

`signals()`返回一个 _信号集合_ ，即一个结构体，将所有信号作为命名方法暴露：  

```rust
// 生成的代码（$为占位符，实际名称取决于实现）：  
impl $SignalCollection {  
    fn damage_taken(&mut self) -> $Signal {...}  
}  

#[godot_api]  
impl INode3D for Monster {  
    fn ready(&mut self) {  
        let sig = self.signals().damage_taken();  
    }  
}  
```

`damage_taken()`方法返回一个自定义生成的 _信号类型_ （代码片段中称为`$Signal`），其API针对`fn damage_taken(amount: i32)`的签名定制。每个`#[signal]`属性都会生成一个独特的信号类型。  

信号类型由实现定义。除了`#[signal]`特定的自定义API外，它还通过`Deref/DerefMut`以[`TypedSignal`][api-typedsignal]为目标，这意味着你可以在每种信号类型上使用所有 _这些_ 方法。  

```admonish note title="信号 API 的可用性"
要让类型信号可用，你需要：

- 一个 `#[godot_api] impl MyClass {}` 块。  
  - 这必须是一个固有 impl，`I*` trait 的 `impl` 并不足够。  
  - 有必要，可以让该 impl 保持为空。  
- 一个 `Base<T>` 字段。

信号（无论是否类型化）**不能**在次级 `impl` 块中声明（那些带有 `#[godot_api(secondary)]` 属性的 `impl` 块）。
```


## 连接信号  

`godot-rust`提供了多种连接信号的方式，具体取决于handler函数的位置。  


### 信号与handler函数在同一对象（self）

将信号连接到同一类的方法很常见。可以通过`connect_self()`方法实现，只需将方法指针作为参数传递：  

```rust
impl Monster {  
    fn on_damage_taken(&mut self, amount: i32) {  
        ... // 更新血条、播放音效等。  
    }  
}  

#[godot_api]  
impl INode3D for Monster {  
    fn ready(&mut self) {  
        self.signals()  
            .damage_taken()  
            .connect_self(Self::on_damage_taken);  
    }  
}  
```

注意`on_damage_taken`没有`#[func]`属性，其所在的`impl`块也没有`#[godot_api]`过程宏。信号接收者是普通的Rust函数！你可以将它们完全隐藏于Godot，仅通过信号访问。  

由于`connect_self()`的参数本质上是`impl FnMut(&mut Self, i32)`，你也可以传递闭包：

```rust
#[godot_api]  
impl INode3D for Monster {  
    fn ready(&mut self) {  
        self.signals()  
            .damage_taken()  
            .connect_self(|this, amount| {
                //         ^^^^  ^^^^^^
                //               类型可推断&mut Self, i32。  
                 
                this.update_healthbar(amount);
                this.play_sound(Sfx::MonsterAttacked);
            });
    }
}
```  

```admonish note title="一次只有一个信号"
如果你想连接多个信号，需要重复调用 `self.signals()`。  
你不能把它存储在一个变量里重复使用。
```


### handler函数在其他对象

如果handler函数需要在`self`以外的对象上运行，可以使用`connect_other()`，它以`&Gd<T>`作为第一个参数：  

```rust
#[godot_api]  
impl INode3D for Monster {  
    fn ready(&mut self) {  
        // 假设伤害被转移到护盾对象。  
        // 该对象存储为字段 `shield: OnReady<Gd<Shield>>`。  
        // &*self.shield 即我们需要的 `&Gd<Shield>`。  

        self.signals()  
            .damage_taken()  
            .connect_other(&*self.shield, Shield::on_damage_taken);  
    }  
}  
```  


### 无对象的handler（关联/静态函数）  

如果handler函数不需要访问`self`，只需使用`connect()`：  

```rust
impl Monster {  
    // 现在是关联函数，不再是方法。  
    fn on_damage_taken(amount: i32) {  
        // 不修改对象本身，但更新某些全局统计。  
    }  
}  

#[godot_api]  
impl INode3D for Monster {  
    fn ready(&mut self) {  
        self.signals()  
            .damage_taken()  
            .connect(Self::on_damage_taken);  

        // 或使用闭包：  
        self.signals()  
            .damage_taken()  
            .connect(|amount| {  
                // 更新全局数据。  
            });  
    }  
}  
```  


## 发射信号  

我们已经看到`#[signal]`属性生成的信号类型提供了多个方法：`connect()`、`connect_self()`和`connect_other()`。同样的信号类型也提供了`emit()`方法，用于触发信号：

```rust
impl Monster {  
    // 可以被其他游戏系统调用。  
    pub fn deal_damage(&mut self, amount: i32) {  
        self.hitpoints -= amount;  
        self.signals().damage_taken().emit(amount);  
    }  
}  
```  

与`connect*()`方法一样，`emit()`是完全类型安全的。你只能传递一个`i32`。如果更新信号定义（例如改为接受`bool`或`enum`表示伤害类型），编译器会捕获所有`connect*`和`emit`调用。重构后你可以安心睡觉。  

`emit()`的另一个优点是它带有参数名称（如`#[signal]`属性中提供的）。这让IDE能提供更多上下文，例如在`emit()`调用中显示参数内联提示。参数类型使用 [`AsArg<T>` trait][api-asarg]，它遵循引擎 API，并在参数类型上提供灵活性。
例如，`"string"` 可以作为 `impl AsArg<GString>` 传入。

除了特定的`emit()`方法外，`TypedSignal`（自定义信号类型的deref目标）还提供了一个通用方法`emit_tuple()`，它接受所有参数的元组（按值传递）。这很少需要，但在你想将多个参数作为“捆绑包”传递时很有用。为了完整起见，上述调用等价于：  

```rust
self.signals().damage_taken().emit_tuple((amount,));  
```  


## 在类外部访问信号  

随着游戏交互的增多，你可能不仅想在`impl Monster`块中配置或发射信号，还想在代码库的其他部分操作。
trait方法 [`WithUserSignals::signals()`][api-withusersignals]允许通过`&mut self`直接访问，但在外部你通常只有`Gd<Monster>`。技术上你可以`bind_mut()`该对象，但有更好的方式无需借用检查。  

因此，`Gd`本身[_也_ 提供了`signals()`方法][api-gd-signals]，返回完全相同的 _信号集合_ API：  

```rust
let monster: Gd<Monster> = ...;  
let sig = monster.signals().damage_taken();  
```  


## Godot 内置信号

Godot 提供了许多内置信号，用于挂接到生命周期和事件中。所有引擎提供的类都实现了
[`WithSignals`][api-withsignals] trait，它是 [`WithUserSignals`][api-withusersignals] 的super trait。

每个类 `T` 都有其自己的信号集合，可以通过 `Gd<T>::signals()` 访问。和类方法一样，信号是可继承的，因此你可以这样做：

```rust
// tree_entered 是 Node 上声明的信号。
let node: Gd<Node> = ...;
let sig = node.signals().tree_entered();

// 你也可以从派生类访问它。
let node: Gd<Node3D> = ...;
let sig = node.signals().tree_entered();
```

这在用户自定义类中同样有效。这意味着我们可以扩展之前的 ready() 实现来连接 Godot 信号：

```rust
#[godot_api]
impl INode3D for Monster {
    fn ready(&mut self) {
        // 之前的代码。
        self.signals()
            .damage_taken()
            .connect_other(&*self.shield, Shield::on_damage_taken);
        
        // 连接到 `Node::renamed` 信号，该信号在节点名称变化时触发。
        self.signals()
            .renamed()
            .connect_self(|this| {
                let new_name = this.base().get_name();
                println!("Monster node renamed to {new_name}.");
            });
    }
}
```

```admonish tip title="禁用类型信号"
为类型信号生成的 API 即使不使用通常也不会造成问题。  
然而，可以通过以下方式禁用代码生成：  

~~~rust
#[godot_api(no_typed_signals)]
impl MyClass { ... }
~~~

这样依然可以使用 `#[signal]`，并会注册每一个这样声明的信号，  
但不会生成一个 `signals()` 集合。
```


### 信号可见性

与Rust中的所有项一样，信号默认是私有的（即仅在其模块和子模块中可见）。
你可以通过向`#[signal]`属性添加`pub`来公开它们：  

```rust
#[godot_api]  
impl Monster {  
    #[signal]  
    pub fn damage_taken(amount: i32);  
}  
```  

当然，`pub(crate)`、`pub(super)`或`pub(in path)`也可以用于更精细的控制。  

```admonish warning title="可见性超出限制"
`#[signal]`的可见性不能超过类的可见性。  

如果出现“无法泄露私有类型”等错误，说明你违反了此规则。  
```

因此，如果你的类声明为`struct Monster`（私有），则不能将信号声明为`pub`或`pub(crate)`。这是由于信号是单独的类型，其API中引用了类类型。将它们“比类更公开”会绕过Rust的隐私规则。  

从语义上讲，这是合理的：唯一需要外部访问的情况是通过`Gd<SomeClass>::signals()`，而这意味着`SomeClass`在该点是可见的。但与`fn`等其他Rust项不同，更宽的可见性不会自动限制为“最多结构体可见性”，而是会导致编译错误。  

注意，你无法分离`connect`和`emit` API的可见性。如果想确保外部只能发射，请将信号保持私有，并在类中提供一个公共包装函数，将调用转发给信号。  


### 从外部连接

假设你有一个音效系统，应在怪物受到伤害时播放音效。你可以从那里连接到信号：  

```rust
impl SoundSystem {
    fn connect_sound_system(&self, monster: &Gd<Monster>) {
        monster.signals()
            .damage_taken()
            .connect_other(self, |this, _amount| {
                this.play_sound(Sfx::MonsterAttacked);
            });
    }
}
```  


### 从外部发射

与连接一样，发射也可以通过`Gd::signals()`进行。其余部分保持不变。  

```rust
fn load_map() {  
    // 所有加载操作。  
    ...  

    // 通知玩家周围世界已加载。  
    let player: Gd<Player> = ...;  
    player.signals().on_world_loaded().emit();  
}  
```  


## 高级信号配置  

`TypedSignal::connect*()`方法设计为直观且覆盖常见用例。如果需要更高级的设置，[`TypedSignal::builder()`][api-typedsignal-builder]提供了高度定制化。  

返回的[`ConnectBuilder`][api-connectbuilder]提供了多个维度的配置：  

- 接收者：`function(args)`、`method(&mut T, args)`、`method(Gd<T>, args)`  
- 提供的对象：无、`self`或其他实例
- 连接标志：`DEFERRED`、`ONESHOT`、`PERSIST`  
- 单线程（默认）或跨线程 ("sync")

一些示例配置：  

```rust
// Connect -> Self::log_event(&mut self, event: String)
signal.builder()
    .flags(ConnectFlags::DEFERRED | ConnectFlags::ONESHOT)
    .connect_self_mut(Self::log_event); // receive &mut self

// Connect -> Logger::log_event(&mut self, event: String)
signal.builder()
    .connect_other_mut(some_gd, Logger::log_event);

// Connect -> Logger::log_event(this: Gd<Self>, event: String)
signal.builder()
    .connect_other_gd(some_gd, Logger::log_event);

// Connect -> Logger::log_thread_safe(event: String)
signal.builder()
    .connect_sync(Logger::log_thread_safe); // associated fn, no receiver
```

构建器方法需要按正确顺序（“阶段”）调用。详见[API文档][api-typedsignal-builder] 。  


### 无类型信号  

Godot 处理无类型信号的低级API仍然可用：  

- [`Object::connect()`][api-object-connect], `Object::connect_ex()`
- [`Object::emit_signal()`][api-object-emitsignal]
- [`Signal::connect()`][api-signal-connect]
- [`Signal::emit()`][api-signal-emit]

新的类型化信号 API 应该能够覆盖全部功能，但在某些情况下，信息只会在运行时才可用，这时无类型的反射 API 就很适合。未来我们也可能将两者结合使用。

要发射一个无类型信号，你可以通过访问基类（以可变方式）调用 `Object::emit_signal` 方法：
结合之前示例中的 `Monster` 结构体，你可以这样发射它的信号：

```rust
self.base_mut().emit_signal(
    "damage_taken",
    &[amount_damage_taken.to_variant()]
);
```

某些类型信号功能仍在计划中，将使信号处理更加流畅。其他功能可能不会移植到godot-rust，例如`Callable::bind()`等效的Rust方法。直接使用闭包即可。  


## 结论  

本章中，我们看到了`godot-rust`的**类型安全信号**如何提供直观且健壮的方式处理Godot的观察者模式，并避免GDScript的某些陷阱。
Rust函数引用或闭包可以直接连接到信号，发射信号通过常规函数调用实现。


[api-asarg]: https://godot-rust.github.io/docs/gdext/master/godot/meta/trait.AsArg.html
[api-withsignals]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithSignals.html
[api-withusersignals]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithUserSignals.html
[api-gd-signals]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html#method.signals
[godot-gdscript-signals]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#signals
[api-typedsignal]: https://godot-rust.github.io/docs/gdext/master/godot/register/struct.TypedSignal.html
[api-typedsignal-builder]: https://godot-rust.github.io/docs/gdext/master/godot/register/struct.TypedSignal.html#method.builder
[api-connectbuilder]: https://godot-rust.github.io/docs/gdext/master/godot/register/struct.ConnectBuilder.html
[api-object-connect]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Object.html#method.connect
[api-object-emitsignal]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Object.html#method.emit_signal
[api-signal-connect]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/struct.Signal.html#method.connect
[api-signal-emit]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/struct.Signal.html#method.emit
