# godot-rust中文文档

godot-rust中文文档是对**gdext**[用户指南][godot-rust-book]的翻译，gdext是 Godot 4的Rust绑定。它涵盖了大量的概念，并且补充了 [API 文档][gdext-docs]。对于 Godot 3，可以参考 [gdnative-book]。

> [!Tip]
> The book is deployed at **[godot-rust.github.io/book][book-web]**.


## Local setup

本书是使用 [mdBook] 和插件 [mdbook-toc]、[mdbook-admonish] 构建的。要安装它们并在本地构建本书，可以运行以下命令：

```bash
cargo install mdbook mdbook-toc mdbook-admonish
mdbook build
```

要在编辑本书时运行带有自动更新的本地服务器，请使用：

```bash
mdbook serve --open
```


### 格式化与拼写检查

[markdownlint] 强制对 Markdown 文件应用一致的风格。它会在 CI 过程中自动运行，但如果你有 `npm` 工具链，你也可以在本地运行它：

```bash
npm install --global markdownlint-cli2
./lint.sh

# To fix certain errors directly:
./lint.sh fix
```


### Oxipng

我们使用 [oxipng] 来优化图像文件的大小。你可以通过 `cargo install oxipng` 安装它，然后按如下方式运行：

```bash
oxipng --strip safe --alpha -r src
```


## 贡献

该仓库仅用于文档。如果需要更改库本身，请在 [主仓库][gdext] 中打开拉取请求和问题，并阅读 [贡献指南][gdext-contribute]。


## 许可

与 gdext 本身一样，gdext 手册也使用 [MPL 2.0][mpl] 许可证。

[book-web]: https://godot-rust.github.io/book
[gdext]: https://github.com/godot-rust/gdext
[godot-rust-book]: https://godot-rust.github.io/book
[gdext-docs]: https://godot-rust.github.io/docs/gdext/master/godot
[gdext-contribute]: https://github.com/godot-rust/gdext/blob/master/Contributing.md
[gdnative-book]: https://github.com/godot-rust/gdnative-book
[markdownlint]: https://github.com/DavidAnson/markdownlint
[mdbook-admonish]: https://github.com/tommilligan/mdbook-admonish
[mdbook-toc]: https://github.com/badboy/mdbook-toc
[mdBook]: https://github.com/rust-lang-nursery/mdBook
[mpl]: https://www.mozilla.org/en-US/MPL
[oxipng]: https://github.com/shssoichiro/oxipng
