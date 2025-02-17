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

- Boolean: `bool`
- Numeric: `int`, `float`

**复合类型**

- Variant (able to hold anything): `Variant`
- String types: `String`, `StringName`, `NodePath`
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

Godot 提供 三种 string 类型: `String` ([`GString`][api-gstring] in Rust), [`StringName`][api-stringname], 和 [`NodePath`][api-nodepath].
`GString` is used as a general-purpose string, while `StringName` is often used for identifiers like class or action names.
The idea is that `StringName` is cheap to construct and compare.[^string-name-Rust]

When working with Godot APIs, you can pass references to the parameter type (e.g. `&GString`), as well as Rust strings `&str`, and `&String`.
To convert different string types in argument contexts (e.g. `StringName` -> `GString`), you can call `arg()`.

```rust
// Label::set_text() takes impl AsArg<GString>.
label.set_text("my text");
label.set_text(&string);           // Rust String
label.set_text(&gstring);          // GString
label.set_text(string_name.arg()); // StringName
```

Outside argument contexts, the `From` trait is implemented for string conversions: `GString::From("my string")`, or `"my_string".into()`.

`StringName` in particular provides a direct conversion from C-string literals such as `c"string"`, [introduced in Rust 1.77][rust-c-strings].
This can be used for _static_ C-strings, i.e. ones that remain allocated for the entire program lifetime. Don't use them for short-lived ones.

## Arrays 和 dictionaries

Godot's linear collection type is [`Array<T>`][api-array]. It is generic over its element type `T`, which can be one of the supported Godot types
(generally anything that can be represented by `Variant`). A special type `VariantArray` is provided as an alias for `Array<Variant>`, which is
used when the element type is dynamically typed.

[`Dictionary`][api-dictionary] is a key-value store, where both keys and values are `Variant`. Godot currently does not support generic
dictionaries, although this feature is [under discussion][godot-generic-dicts].

Arrays and dictionaries can be constructed using three macros:

```rust
let a = array![1, 2, 3];          // Array<i64>
let b = varray![1, "two", true];  // Array<Variant>
let c = dict!{"key": "value"};    // Dictionary
```

Their API is similar, but not identical to Rust's standard types `Vec` and `HashMap`. An important difference is that `Array` and `Dictionary`
are reference-counted, which means that `clone()` will not create an independent copy, but another reference to the same instance. Furthermore,
since internal elements are stored as variants, they are not accessible by reference. This is why the `[]` operator (`Index/IndexMut` traits)
is absent, and `at()` is provided instead, returning by value.

```rust
let a = array![0, 11, 22];

assert_eq!(a.len(), 3);
assert_eq!(a.at(1), 11);         // Panics on out-of-bounds.
assert_eq!(a.get(1), Some(11));  // Also by value, not Some(&11).

let mut b = a.clone();   // Increment reference-count.
b.set(2, 33);            // Modify new ref.
assert_eq!(a.at(2), 33); // Original array has changed.

b.clear();
assert!(b.is_empty());
assert_eq!(b, Array::new()); // new() creates an empty array.
```

```rust
let c = dict! {
    "str": "hello",
    "int": 42,
    "bool": true,
};

assert_eq!(c.len(), 3);
assert_eq!(c.at("str"), "hello".to_variant());    // Panics on missing key.
assert_eq!(c.get("int"), Some(42.to_variant()));  // Option<Variant>, again by value.

let mut d = c.clone();            // Increment reference-count.
d.insert("float", 3.14);          // Modify new ref.
assert!(c.contains_key("float")); // Original dict has changed.
```

To iterate, you can use `iter_shared()`. This method works almost like `iter()` on Rust collections, but the name highlights that you do not
have unique access to the collection during iteration, since there might exist another reference to the collection. This also means it's your
responsibility to ensure that the array/dictionary is not modified in unintended ways during iteration (which should be safe, but may lead to
data inconsistencies).

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

[`Packed*Array`][api-packed-array] types are used for storing elements space-efficiently ("packed") in contiguous memory.
The `*` stands for the element type, e.g. `PackedByteArray` or `PackedVector3Array`.

```rust
// Create from slices.
let bytes = PackedByteArray::from(&[0x0A, 0x0B, 0x0C]);
let ints = PackedInt32Array::from(&[1, 2, 3]);

// Get/set individual elements using Index and IndexMut operators.
ints[1] = 5;
assert_eq!(ints[1], 5);

// Access as Rust shared/mutable slices.
let bytes_slice: &[u8] = b.as_slice();
let ints_slice: &mut [i32] = i.as_mut_slice();

// Access sub-ranges of the array using the same type.
let part: PackedByteArray = bytes.subarray(1, 3); // 1..3, or 1..=2
assert_eq!(part.as_slice(), &[0x0B, 0x0C]);
```

Unlike `Array`, packed arrays use copy-on-write instead of reference counting. When you clone a packed array, you get a new independent instance.
Cloning is cheap as long as you don't modify either instance. Once you use a write operation (anything with `&mut self`), the packed array will
allocate its own memory and copy the data.

<br>

---

**Footnotes**

[^packed-vec4]: `PackedVector4Array` is only available since Godot version 4.3; added in [PR #85474][godot-packed-vector4].

[^num-types]: Godot's `int` and `float` types are canonically mapped to `i64` and `f64` in Rust. However, some Godot APIs specify the domain of
these types more specifically, so it's possible to encounter `i8`, `u64`, `f32` etc.

[^str-types]: String types `GString`, `StringName`, and `NodePath` can be passed into Godot APIs as string literals, hence the `"string"` syntax
in this example. To assign to your own value, e.g. of type `GString`, you can use `GString::from("string")` or `"string"`.

[^string-name-Rust]: When constructing `StringName` from `&str` or `String`, the conversion is rather expensive, since UTF-8 is re-encoded as
UTF-32. As Rust recently introduced C-string literals (`c"hello"`), we can now directly construct from them in case of ASCII. This is more
efficient, but keeps memory allocated until shutdown, so don't use it for rarely used temporaries.
See [API docs][api-stringname] and [issue #531][issue-stringname-perf] for more information.


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
