# Windows + MinGW + VSCode + clang + clangd + CMake + Ninja + LLDB

***[@Bachop](https://github.com/Bachop)***

本文档基于使用 **VSCode** 作为代码/文本编辑器，**clangd** 作为语言服务器，**CMake** 作为构建系统，**Ninja** 作为构建生成器，**LLDB** 作为调试工具的需求编写，适用于 **Windows** 系统下使用 **MinGW** 工具链的 **C / C++** 开发。

> 本配置流程同样适用于 ***Cursor***.

---

## 1. 安装必要工具

### 1.1 编译器 (以 MSYS2 MinGW 为例)

1. 下载安装 **[MSYS2](https://www.msys2.org/)**；
2. 打开 **MSYS2 UCRT64** 终端，执行：
   ```bash
   pacman -Syu
   pacman -S mingw-w64-ucrt-x86_64-clang \
             mingw-w64-ucrt-x86_64-clang-tools-extra \
   ```
3. 将 `D:\msys64\ucrt64\bin` 添加到系统 **PATH** 环境变量中（路径根据实际安装位置调整）。

### 1.2 构建工具
- **[CMake](https://cmake.org/download/)**
- **[Ninja](https://ninja-build.org/download.html)**（或使用命令安装）
    ```
    winget install Ninja-build.Ninja
    ```

### 1.3 编辑器、扩展/插件
- **[VSCode](https://code.visualstudio.com/download)** / **[Cursor](https://cursor.com/download)** (文本编辑器)
- **[clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd)** (必须)
- **[CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)** (用于调试)
- **可选**：**[CMake Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools)**（可能与手动配置冲突，建议二选一）
- **禁用**： **[Microsoft C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)** 扩展的 **IntelliSense**（避免冲突）

### 1.3 验证安装

在终端中执行：
```bash
clang --version
clangd --version
cmake --version
ninja --version
```

---

## 2. 创建项目结构

```
your_project/
├── .vscode/
│   ├── settings.json
│   ├── tasks.json
│   └── launch.json
├── inc/
│   └── header.h
├── src/
│   ├── mainn.cpp
│   └── source.cpp
├── CMakeLists.txt
└── (build/ 目录将在配置时自动生成)
```

---

## 3. 编写 `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.10)
project(your_project LANGUAGES C CXX)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(SOURCES
    ./src/main.cpp    # 列举源文件
)

add_executable(your_project ${SOURCES})

target_include_directories(your_project PRIVATE include)
```

> `your_project` 将用于调试配置中的 `program` 字段。

---

## 4. 配置 VSCode 设置文件

### `.vscode/settings.json`

```json
{
    "clangd.arguments": [
        "--compile-commands-dir=build",           // 读取 build/compile_commands.json
        "--query-driver=D:/msys64/ucrt64/bin/clang.exe",    //切换成实际路径
        "--query-driver=D:/msys64/ucrt64/bin/clang++.exe",  //同上
        "--background-index",
        "--clang-tidy",
        "--header-insertion=iwyu"
    ],

    "clangd.path": "D:/msys64/ucrt64/bin/clangd.exe"        // 避免与 clangd 冲突
}
```

> 如果 `--query-driver` 不能自动解析，可以在项目根目录创建 `.clangd` 文件手动添加 `-isystem` 路径（见附录）。

---

## 5. 配置构建任务

### `.vscode/tasks.json`

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "type": "shell",
            "label": "cmake-configure",
            "command": "cmake",
            "args": [
                "-B", "build",
                "-S", ".",
                "-G", "Ninja",
                "-DCMAKE_BUILD_TYPE=Debug",
                "-DCMAKE_EXPORT_COMPILE_COMMANDS=ON"
                "-DCMAKE_C_COMPILER=D:/msys64/ucrt64/bin/clang.exe",    // 显式指定编译器，切换成实际路径
                "-DCMAKE_CXX_COMPILER=D:/msys64/ucrt64/bin/clang++.exe" //同上
            ],
            "group": "build",
            "problemMatcher": []
        },
        {
            "type": "shell",
            "label": "cmake-build",
            "command": "cmake",
            "args": [
                "--build", "build",
                "--config", "Debug",
                "-j", "4"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "dependsOn": ["cmake-configure"],    // 构建前自动配置
            "problemMatcher": ["$msCompile"]    // 使用 MSVC 编译器的错误匹配器，适用于 Windows
        }
    ]
}
```

- `Ctrl+Shift+B` 会执行 `cmake-build`（默认构建任务），并自动先执行 `cmake-configure`。
- `compile_commands.json` 将生成在 `build/` 目录。

---

## 6. 配置调试器 (launch.json)

### 使用 CodeLLDB (推荐配合 clangd)

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug with LLDB",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/build/Ninja_Debug/your_project.exe",
            "args": [],
            "cwd": "${workspaceFolder}",
            "preLaunchTask": "cmake-build",
            "terminal": "integrated"
        }
    ]
}
```

> - `program` 路径必须与 `CMakeLists.txt` 中的 `add_executable` 目标名一致（Windows 下加 `.exe`）。
> - `preLaunchTask` 的值必须与 `tasks.json` 中的构建任务 `label` 完全一致。

### 备选：使用 GDB (如果 LLDB 遇到问题)

```json
{
    "name": "Debug with GDB",
    "type": "cppdbg",
    "request": "launch",
    "program": "${workspaceFolder}/build/your_project.exe",
    "args": [],
    "stopAtEntry": false,
    "cwd": "${workspaceFolder}",
    "environment": [],
    "externalConsole": true,
    "MIMode": "gdb",
    "miDebuggerPath": "D:/msys64/ucrt64/bin/gdb.exe",   //切换成实际路径
    "setupCommands": [
        {
            "description": "Enable pretty-printing",
            "text": "-enable-pretty-printing",
            "ignoreFailures": true
        }
    ],
    "preLaunchTask": "cmake-build"
}
```

---

## 7. 标准工作流程

### 7.1 首次配置（生成 `compile_commands.json`）

- 在 VSCode 中打开项目根目录
- 按 `Ctrl+Shift+B` 执行构建任务  
  → 自动完成 CMake 配置 + 编译  
  → 生成 `build/compile_commands.json`

### 7.2 编写代码

- clangd 提供代码补全、错误提示、跳转到定义等智能功能。

### 7.3 调试

1. 在 `main.cpp` 中设置断点（点击行号左侧）。
2. 按 `F5` 开始调试。  
   → 自动执行 `cmake-build`（增量编译）  
   → 启动 LLDB，停在断点处。  
   → 可查看变量、单步执行、监视表达式。

### 7.4 仅运行（不调试）

在集成终端中执行：
```bash
./build/your_project.exe
```

---

## 8. 常见问题与解决方案

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| clangd 找不到 `iostream` 等标准库 | clangd 无法自动获取 MinGW 标准库路径 | 在 `settings.json` 中添加 `--query-driver` 指向 `clang.exe/clang++.exe`|
| 按 F5 提示“找不到任务 CMake Build” | `launch.json` 中的 `preLaunchTask` 与 `tasks.json` 的 `label` 不匹配 | 确保字符串完全一致，例如 `"cmake-build"` |
| 调试时程序立即退出（未停在断点） | 未设置断点，或 `stopAtEntry` 为 `false` | 在 `main` 函数开头设置断点，或设置 `"stopAtEntry": true` |
| 断点变灰（不可用） | 编译时未生成调试信息 | 确保 CMake 配置中 `-DCMAKE_BUILD_TYPE=Debug` |
| LLDB 无法正确调试 MinGW 程序 | LLDB 对 Windows/MinGW 支持有限 | 改用 GDB（`type: cppdbg` + `miDebuggerPath`） |
| `cmake-configure` 每次都重新运行 | 希望跳过配置步骤 | 移除 `dependsOn`，手动运行一次 `cmake-configure` |

---

## 9. 总结

该工作流程的核心组件与交互关系：

- **clangd** 依赖 `compile_commands.json` 理解代码结构
- **CMake** 生成该文件并作为构建系统
- **Ninja** 作为后端构建工具，速度快
- **CodeLLDB** 提供调试能力
- **VSCode tasks.json** 将构建集成到快捷键和调试前步骤

配置完成后，可以在 VSCode 中获得接近 IDE 的 C++ 开发体验，兼具轻量与可移植性。

---

**[Version：1.0](#-Windows-+-MinGW-+-VSCode-+-clang-+-clangd-+-CMake-+-Ninja-+-LLDB)***, Created by [@Bachop](https://github.com/Bachop), 9th May, 2026*