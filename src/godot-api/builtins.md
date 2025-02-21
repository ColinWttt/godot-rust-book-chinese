<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 内置类型（Built-in types）

所谓的“内置类型”或简称“内置”（builtins）是 Godot 提供的基本类型。值得注意的是，这些并不是 _类_（_classes_）。
另请参见 [Godot 中的基本内置类型][godot-docs-builtins]。


## 目录
<!-- toc -->

## 类列清单

以下是按类别列出的所有内置类型的完整清单。我们使用 GDScript 中的名称；在下面，我们将解释它们如何映射到 Rust。

**简单类型**

- 布尔: `bool`
- 数值: `int`, `float`

**复合类型**

- Variant (able to hold anything): `Variant`
- String 类型: `String`, `StringName`, `NodePath`
- Ref-counted containers: `Array` (`Array[T]`), `Dictionary`
- Packed arrays: `Packed*Array` for following element types:  
  `Byte`, `Int32`, `Int64`, `Float32`, `Float64`, `Vector2`, `Vector3`, `Vector4`[^packed-vec4], `Color`, `String`
- Functional: `Callable`, `Signal`

**几何类型**

- Vectors: `Vector2`, `Vector2i`, `Vector3`, `Vector3i`, `Vector4`, `Vector4i`
- Bounding boxes: `Rect2`, `Rect2i`, `AABB`
- Matrices: `Transform2D`, `Transform3D`, `Basis`, `Projection`
- Rotation: `Quaternion`
- Geometric objects: `Plane`

**杂项**

- Color: `Color`
- Resource ID: `RID`


### Rust 映射

gdext API 中的 Rust 类型尽可能以最接近的方式表示相应的 Godot 类型。例如，它们被用作 API 函数的参数和返回类型。它们可以通过 `godot::builtin` 访问，且大多数符号也包含在 `prelude` 中。

大多数内置类型都有 1:1 的对应关系（例如:`Vector2f`、`Color` 等）。以下列表显示了一些值得注意的映射：

| GDScript 类型             | Rust 类型                             | Rust 示例表达      |
|---------------------------|---------------------------------------|-------------------------------|
| `int`                     | `i64`[^num-types]                     | `-12345`                      |
| `float`                   | `f64`[^num-types]                     | `3.14159`                     |
| `real`                    | `real` (either `f32` or `f64`)        | `real!(3.14159)`              |
| `String`                  | `GString`                             | `"Some string"` [^str-types]  |
| `StringName`              | `StringName`                          | `"MyClass"` [^str-types]      |
| `NodePath`                | `NodePath`                            | `"Nodes/MyNode"` [^str-types] |
| `Array[T]`                | `Array<T>`                            | `array![1, 2, 3]`             |
| `Array`                   | `VariantArray`<br>or `Array<Variant>` | `varray![1, "two", true]`     |
| `Dictionary`              | `Dictionary`                          | `dict!{"key": "value"}`       |
| `AABB`                    | `Aabb`                                | `Aabb::new(pos, size)`        |
| `Object`                  | `Gd<Object>`                          | `Object::new_alloc()`         |
| `SomeClass`               | `Gd<SomeClass>`                       | `Resource::new_gd()`          |
| `SomeClass` (nullable)    | `Option<Gd<SomeClass>>`               | `None`                        |
| `Variant` (also implicit) | `Variant`                             | `Variant::nil()`              |

请注意，Godot 的类 API 目前尚未提供空值信息。这意味着我们必须保守地假设对象可能为 null，因此在对象返回类型上使用 `Option<Gd<T>>` 而不是 `Gd<T>`。这通常会导致不必要的解包操作。

可空类型（nullable types）正在 Godot 方面进行研究 [关于 Godot 端的空值问题][godot-nullability-issue]。如果上游暂时没有解决方案，我们可能会考虑自己的变通方法，但这可能需要对许多 API 进行手动注解。


## String 类型

Godot 提供 三种 string 类型: `String` (在 Rust 中是[`GString`][api-gstring]), [`StringName`][api-stringname], 和 [`NodePath`][api-nodepath]。
`GString` 用作通用字符串，而 `StringName` 通常用于标识符，例如类名或动作名称。`StringName` 的特点是构造和比较非常高效。[^string-name-Rust]

在使用 Godot API 时，你可以将参数类型的引用传递给函数（例如 `&GString`），以及 Rust 字符串 `&str` 和 `&String`。
要在参数上下文中转换不同的字符串类型（例如 `StringName` -> `GString`），你可以调用 `arg()`。

```rust
// Label::set_text() takes impl AsArg<GString>.
label.set_text("my text");
label.set_text(&string);           // Rust String
label.set_text(&gstring);          // GString
label.set_text(string_name.arg()); // StringName
```

在参数上下文之外，`From` trait 用于字符串转换：`GString::From("my string")`，或者使用 `"my_string".into()`。

特别是，`StringName`提供了从C字符串字面量（如`c"string"`）的直接转换，[该特性在Rust 1.77中引入][rust-c-strings]。
这可以用于 _静态_ C字符串，即那些在整个程序生命周期内保持分配的字符串。不要将其用于短生命周期的字符串。


## 数组和字典

Godot的线性集合类型是 [`Array<T>`][api-array]. 它是对元素类型`T`的泛型，可以是任何支持的Godot类型（通常是可以由`Variant`表示的任何类型）。
提供了一个特殊类型`VariantArray`，作为`Array<Variant>`的别名，当元素类型是动态类型时使用。


[`Dictionary`][api-dictionary] 是一个键值对存储，其中键和值都是`Variant`。Godot目前不支持泛型字典，尽管这个特性正在[讨论中][godot-generic-dicts].

数组和字典可以使用三个宏来构建：

```rust
let a = array![1, 2, 3];          // Array<i64>
let b = varray![1, "two", true];  // Array<Variant>
let c = dict!{"key": "value"};    // Dictionary
```

它们的API类似，但与Rust的标准类型`Vec`和`HashMap`并不完全相同。一个重要的区别是，`Array`和`Dictionary`是引用计数的，这意味着`clone()`不会创建一个独立的副本，而是另一个对同一实例的引用。
此外，由于内部元素以`variant`存储，它们不能通过引用访问。这就是为什么缺少`[]`操作符（`Index/IndexMut`traits，而是提供了`at()`方法，它返回的是值。

```rust
let a = array![0, 11, 22];

assert_eq!(a.len(), 3);
assert_eq!(a.at(1), 11);         // 超出边界会panic。
assert_eq!(a.get(1), Some(11));  // 也返回值，而不是Some(&11)。

let mut b = a.clone();   // 增加 reference-count.
b.set(2, 33);            // 修改新引用。
assert_eq!(a.at(2), 33); // 原始 array 已经改变。

b.clear();
assert!(b.is_empty());
assert_eq!(b, Array::new()); // new() 创建一个空 array.
```

```rust
let c = dict! {
    "str": "hello",
    "int": 42,
    "bool": true,
};

assert_eq!(c.len(), 3);
assert_eq!(c.at("str"), "hello".to_variant());    // 没找到键会panic。
assert_eq!(c.get("int"), Some(42.to_variant()));  // Option<Variant>，同样是值。

let mut d = c.clone();            // 增加 reference-count.
d.insert("float", 3.14);          // 修改新引用。
assert!(c.contains_key("float")); // 原始字典已经改变。
```

要进行迭代，可以使用`iter_shared()`。这个方法的工作方式几乎与Rust集合的`iter()`相似，但其名称强调了在迭代期间你并没有对集合的唯一访问权，因为可能存在对集合的其他引用。
这也意味着你有责任确保在迭代过程中，数组或字典没有被不必要地修改（虽然应该是安全的，但可能会导致数据不一致）。

```rust
let a = array!["one", "two", "three"];
let d = dict!{"one": 1, "two": 2.0, "three": Vector3::ZERO};

for elem in a.iter_shared() {
    // elem has type GString.
    println!("Element: {elem}");
}

for (key, value) in d.iter_shared() {
    // key and value both have type Variant.
    println!("Key: {key}, value: {value}");
}
```


## Packed arrays

[`Packed*Array`][api-packed-array] 类型用于在连续的内存中高效地存储元素（“打包”）。
`*` 代表元素类型，例如 `PackedByteArray` 或 `PackedVector3Array`.

```rust
// Create from slices.
let bytes = PackedByteArray::from(&[0x0A, 0x0B, 0x0C]);
let ints = PackedInt32Array::from(&[1, 2, 3]);

// 使用Index和IndexMut操作符来获取/设置单个元素。
ints[1] = 5;
assert_eq!(ints[1], 5);

//  作为Rust shared/mutable slices访问。
let bytes_slice: &[u8] = b.as_slice();
let ints_slice: &mut [i32] = i.as_mut_slice();

// 使用相同类型访问数组的子范围。
let part: PackedByteArray = bytes.subarray(1, 3); // 1..3, or 1..=2
assert_eq!(part.as_slice(), &[0x0B, 0x0C]);
```

与`Array`不同，`Packed*Array`使用写时复制（copy-on-write），而不是引用计数。当你克隆一个`Packed*Array`时，你会得到一个新的独立实例。只要不修改任何实例，克隆是便宜的。

一旦使用写操作（任何带有`&mut self`的操作），`Packed*Array`会分配自己的内存并复制数据。

<br>

---

**脚注**

[^packed-vec4]: `PackedVector4Array` 仅在Godot 4.3版本中可用；在 [PR #85474][godot-packed-vector4]中添加。.

[^num-types]: Godot的`int`和`float`类型在Rust中通常映射为`i64`和`f64`。然而，某些Godot API更加具体地指定了这些类型的域，因此可能会遇到`i8`、`u64`、`f32`等类型。

[^str-types]: 字符串类型`GString`、`StringName`和`NodePath`可以作为字符串字面量传递给Godot API，因此在这个示例中使用了`"string"`语法。要为自己的值赋值，例如类型为`GString`，可以使用`GString::from("string")` 或 `"string"`.

[^string-name-Rust]: 当从`&str`或`String`构造`StringName`时，转换相当昂贵，因为UTF-8会被重新编码为UTF-32。由于Rust最近引入了C字符串字面量（`c"hello"`），如果是ASCII字符串，现在我们可以直接从它们构造`StringName`。这种方式会更高效，但是会使内存在程序关闭之前一直保持分配，因此不要将其用于生命周期短暂的临时字符串。有关更多信息，请参阅[API文档][api-stringname] 和 [issue #531][issue-stringname-perf]。


[api-array]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/struct.Array.html
[api-dictionary]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/struct.Dictionary.html
[api-gstring]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/struct.GString.html
[api-nodepath]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/struct.NodePath.html
[api-packed-array]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/index.html#structs
[api-stringname]: https://godot-rust.github.io/docs/gdext/master/godot/builtin/struct.StringName.html
[godot-docs-builtins]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#basic-built-in-types
[godot-generic-dicts]: https://github.com/godotengine/godot/pull/78656
[godot-nullability-issue]: https://github.com/godotengine/godot-proposals/issues/162
[godot-packed-vector4]: https://github.com/godotengine/godot/pull/85474
[issue-stringname-perf]: https://github.com/godot-rust/gdext/issues/531
[rust-c-strings]: https://doc.rust-lang.org/nightly/edition-guide/rust-2021/c-string-literals.html#c-string-literals
