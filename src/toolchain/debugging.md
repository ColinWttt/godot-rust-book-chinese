<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# 调试（Debugging）


用 gdext 编写的扩展可以像调试其他 Rust 程序一样使用 LLDB 进行调试。
主要的区别在于，LLDB 会启动或附加到 Godot 的 C++ 可执行文件：无论是 Godot 编辑器，还是你的自定义 Godot 应用程序。
随后，Godot 会加载你的扩展（它本身是一个动态库），并通过它加载你的 Rust 代码。

启动或附加 LLDB 的过程会根据你的 IDE 和平台有所不同。除非你使用的是 Godot 的调试版本，否则你只会看到 Rust 代码中的栈帧（stack frames）符号。


## 使用 VS Code 启动调试

以下是一个 Visual Studio Code 启动配置的示例。启动配置应添加到 `./.vscode/launch.json` 文件中，并相对于项目根目录进行配置。
这个示例假设你已安装 [CodeLLDB] 扩展，这在 Rust 开发中是常见的。


```json
{
    "configurations": [
        {
            "name": "Debug Project (Godot 4)",
            "type": "lldb", // CodeLLDB 扩展提供的类型
            "request": "launch",
            "preLaunchTask": "rust: cargo build",
            "cwd": "${workspaceFolder}",
            "args": [
                "-e", // 启动编辑器（移除此项以直接启动场景）
                "-w", // 窗口模式
            ],
            "linux": {
                "program": "/usr/local/bin/godot4",
            },
            "windows": {
                "program": "C:\\Program Files\\Godot\\Godot_v4.1.X.exe",
            },
            "osx": {
                // 注意：在 macOS 上，需要手动重新签名 Godot.app 
                // 才能启用调试（见下文）
                "program": "/Applications/Godot.app/Contents/MacOS/Godot",
            }
        }
    ]
}
```


## 在 macOS 上调试

在 macOS 上，将调试器附加到一个未本地编译的可执行文件（例如 Godot 编辑器）时，必须考虑到 _系统完整性保护_（SIP）安全特性。
即使你的扩展是本地编译的，LLDB 也无法在没有手动重新签名的情况下附加到 Godot _主进程_。

为了重新签名，只需创建一个名为 `editor.entitlements` 的文件，并将以下内容添加进去。
确保使用下面的 `editor.entitlements` 文件，而不是 [Godot 文档](https://docs.godotengine.org/en/stable/contributing/development/debugging/macos_debug.html) 中的版本，
因为它包含了当前 Godot 指令中未包含的 `com.apple.security.get-task-allow` 键。


```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist 
  PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>com.apple.security.cs.allow-dyld-environment-variables</key>
        <true/>
        <key>com.apple.security.cs.allow-jit</key>
        <true/>
        <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
        <true/>
        <key>com.apple.security.cs.disable-executable-page-protection</key>
        <true/>
        <key>com.apple.security.cs.disable-library-validation</key>
        <true/>
        <key>com.apple.security.device.audio-input</key>
        <true/>
        <key>com.apple.security.device.camera</key>
        <true/>
        <key>com.apple.security.get-task-allow</key>
        <true/>
    </dict>
</plist>
```

创建此文件后，你可以在终端中运行以下命令以完成重新签名过程：

```bash
codesign -s - --deep --force --options=runtime \
    --entitlements ./editor.entitlements /Applications/Godot.app
```

建议将此文件提交到版本控制中，因为如果你有团队，每个开发者都需要重新签名他们本地的安装。不过，这个过程只需每次安装Godot时执行一次。

[CodeLLDB]: https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb
