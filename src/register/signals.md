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

```GDScript
signal damage_taken  
signal damage_taken(amount)  
signal damage_taken(amount: int)  
```  

然而，上述声明之间的区别仅是信息性的（例如显示在类的文档中）。
来看一个例子：  

```GDScript
signal damage_taken(amount: int)  

func log_damage():  
    print("damaged!")  

func _ready():  
    damage_taken.connect(log_damage)  
```  

注意`log_damage()`没有参数，但你连接它时既不会在解析时也不会在运行时收到警告。  

不止`connect`有这个问题，再看一个向`emit()`传递错误类型参数的例子：  

```GDScript
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

一旦注册了至少一个信号，`godot-rust`会为你的类实现`WithSignals`[api-withsignals] trait。
这提供了`signals()`方法，现在可以在类方法中访问。  
`signals()`返回一个_信号集合_，即一个结构体，将所有信号作为命名方法暴露：  

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

`damage_taken()`方法返回一个自定义生成的_信号类型_（代码片段中称为`$Signal`），其API针对`fn damage_taken(amount: i32)`的签名定制。每个`#[signal]`属性都会生成一个独特的信号类型。  

信号类型由实现定义。除了`#[signal]`特定的自定义API外，它还通过`Deref/DerefMut`以[`TypedSignal`][api-typedsignal]为目标，这意味着你可以在每种信号类型上使用所有_这些_方法。  


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
            .connect_self(|this: &mut Self, amount| {  
                //               ^^^^^^^^^  
                //               必须显式；其他参数类型可推断。  
                 
                ... // 更新血条、播放音效等。  
            });  
    }  
}  
```  


### handler函数在其他对象

如果handler函数需要在`self`以外的对象上运行，可以使用`connect_obj()`，它以`&Gd<T>`作为第一个参数：  

```rust
#[godot_api]  
impl INode3D for Monster {  
    fn ready(&mut self) {  
        // 假设伤害被转移到护盾对象。  
        // 该对象存储为字段 `shield: OnReady<Gd<Shield>>`。  
        // &*self.shield 即我们需要的 `&Gd<Shield>`。  

        self.signals()  
            .damage_taken()  
            .connect_obj(&*self.shield, Shield::on_damage_taken);  
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

我们已经看到`#[signal]`属性生成的信号类型提供了多个方法：`connect()`、`connect_self()`和`connect_obj()`。同样的信号类型也提供了`emit()`方法，用于触发信号：

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

`emit()`的另一个优点是它带有参数名称（如`#[signal]`属性中提供的）。这让IDE能提供更多上下文，例如在`emit()`调用中显示参数内联提示。  

除了特定的`emit()`方法外，`TypedSignal`（自定义信号类型的deref目标）还提供了一个通用方法`emit_tuple()`，它接受所有参数的元组（按值传递）。这很少需要，但在你想将多个参数作为“捆绑包”传递时很有用。为了完整起见，上述调用等价于：  

```rust
self.signals().damage_taken().emit_tuple((amount,));  
```  


## 在类外部访问信号  

随着游戏交互的增多，你可能不仅想在`impl Monster`块中配置或发射信号，还想在代码库的其他部分操作。
trait方法[`WithSignals::signals()`][api-withsignals]允许通过`&mut self`直接访问，但在外部你通常只有`Gd<Monster>`。技术上你可以`bind_mut()`该对象，但有更好的方式无需借用检查。  

因此，`Gd`本身[也提供了`signals()`方法][api-gd-signals]，返回完全相同的_信号集合_API：  

```rust
let monster: Gd<Monster> = ...;  
let sig = monster.signals().damage_taken();  
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
        let this = self.to_gd(); // Gd<SoundSystem>  

        monster.signals()  
            .damage_taken()  
            .connect_obj(this, |s: &mut Self, _amount| {  
                s.play_sound(Sfx::MonsterAttacked);  
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

`TypedSignal::connect*()`方法设计为直观且覆盖常见用例。如果需要更高级的设置，[`TypedSignal::connect_builder()`][api-typedsignal-connectbuilder]提供了高度定制化。  

返回的`ConnectBuilder`提供了多个维度的配置：  

- 接收者：`function(args)`、`method(&self, args)`、`method(&mut self, args)`  
- 提供的对象：无、`&mut self`或`Gd<T>`  
- 连接标志：`DEFERRED`、`ONESHOT`、`PERSIST`  
- 单线程（默认）或跨线程 (_sync_)

通过调用`done()`完成。一些示例配置：  

```rust
// Connect -> Self::log_event(&self, event: String)  
signal.connect_builder()  
    .object_self() // 传入 &self（信号所在的对象）  
    .method_immut(Self::log_event) // 接收 &self  
    .flags(ConnectFlags::DEFERRED | ConnectFlags::ONESHOT)  
    .done();  

// Connect -> Logger::log_event_mut(&mut self, event: String)  
signal.connect_builder()  
    .object(some_gd) // 传入 Gd<T>（任意对象）  
    .method_mut(Logger::log_event_mut) // 接收 &mut self  
    .done();  

// Connect -> Logger::log_event(event: String)  
signal.connect_builder()  
    .function(Logger::log_event) // 关联函数，无接收者  
    .sync() // 允许另一个线程接收信号（不会panic）  
    .done();  
```  

构建器方法需要按正确顺序（“阶段”）调用。详见[API文档][api-typedsignal-connectbuilder]。  


### 无类型信号  

Godot 处理无类型信号的低级API仍然可用：  

- [`Object::connect()`][api-object-connect], `Object::connect_ex()`
- [`Object::emit_signal()`][api-object-emitsignal]
- [`Signal::connect()`][api-signal-connect]
- [`Signal::emit()`][api-signal-emit]

它们可以作为新类型信号API尚未覆盖的领域的后备（例如Godot的内置信号），或在仅有一些运行时信息可用的情况下使用。  

某些类型信号功能仍在计划中，将使信号处理更加流畅。其他功能可能不会移植到`godot-rust`，例如`Callable::bind()`等效的Rust方法。直接使用闭包即可。  


## 结论  

本章中，我们看到了`godot-rust`的**类型安全信号**如何提供直观且健壮的方式处理Godot的观察者模式，并避免GDScript的某些陷阱。
Rust函数引用或闭包可以直接连接到信号，发射信号通过常规函数调用实现。


[api-object]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Object.html
[api-signal]: https://godot-rust.github.io/docs/gdext/master/godot/register/derive.GodotClass.html#signals
[api-withsignals]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithSignals.html
[api-gd-signals]: https://godot-rust.github.io/docs/gdext/master/godot/obj/struct.Gd.html#method.signals
[godot-gdscript-signals]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#signals
[api-typedsignal]: https://godot-rust.github.io/docs/gdext/master/godot/register/struct.TypedSignal.html
[api-typedsignal-connectbuilder]: https://godot-rust.github.io/docs/gdext/master/godot/register/struct.TypedSignal.html#method.connect_builder

[api-object-connect]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Object.html#method.connect
[api-object-emitsignal]: https://godot-rust.github.io/docs/gdext/master/godot/classes/struct.Object.html#method.emit_signal
[api-signal-connect]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/struct.Signal.html#method.connect
[api-signal-emit]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/struct.Signal.html#method.emit
