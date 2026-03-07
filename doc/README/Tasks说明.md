# Tasks.json 文件说明

本文件介绍 Tasks 中的内容以及如何编写 tasks.json 文件

> tasks.json 是 VS Code 的 任务配置文件，位于 .vscode/ 目录下，用来告诉 VS Code 怎样去运行外部命令行程序（比如编译、打包、运行脚本、上传固件……）。

## 本工程 tasks.json 介绍

本项目通过 Tasks 插件，将 tasks.json 中定义的每一个 task 封装为一个个 vscode 底部的按钮，便于操作。

### CMake Configure

这是第一个task，

```json
{
    "label": "CMake Configure",
    "type": "shell",
    "command": "cmake",
    "args": [
        "-DCMAKE_TOOLCHAIN_FILE=${cwd}/mcu/project/arm-gnu-none-eabi.cmake",
        "-DCMAKE_SYSTEM_NAME=Generic",
        "-DCMAKE_EXPORT_COMPILE_COMMANDS:BOOL=TRUE",
        "-GNinja",
        "-S", 
        "mcu/project",
        "-B", 
        "mcu/build"
    ],
    "options": {
        "cwd": "${workspaceFolder}"
    },
    "group": {
        "kind": "build",
        "isDefault": true
    }
}
```

这段 task 会在终端里构造出一条完整的 CMake 配置命令，为项目生成 Ninja 的构建目录。

其中

- "label": "CMake Configure" 
  > VS Code 在任务面板里显示的名字。可以任意取。

- "type": "shell"  
  > 告诉 VS Code，用 shell 来执行这条命令。

- "command": "cmake"  
  > 这就是要执行的程序。这里表示：调用系统里的 CMake。相当于在终端敲 cmake。

- "args": [...]   
  > 这是给 cmake 命令追加的参数列表。

  - "-DCMAKE_TOOLCHAIN_FILE=${cwd}/mcu/project/arm-gnu-none-eabi.cmake"
  
    > 告诉 CMake：使用指定的 交叉编译 toolchain 文件，即使用等号后面路径的文件里的要求来编译文件。

    > `${cwd}` 是 VS Code 在任务运行时的 “current working directory”（当前目录）。这里等价于 `${workspaceFolder}` 。

  - "-DCMAKE_SYSTEM_NAME=Generic"
  
    > 告诉 CMake，目标环境而是一个无操作系统的 MCU 环境。

  - "-DCMAKE_EXPORT_COMPILE_COMMANDS:BOOL=TRUE"
  
    > 让 CMake 导出一个 compile_commands.json。这个文件非常有用：clangd 或 vscode C/C++ 扩展 用它来知道你的 include、宏定义、编译命令，从而：代码跳转更准确、自动补全更智能、错误提示更真实

  - "-GNinja" 
  
    > 表示之后构建会用：**ninja** ，而不是 makefile。

  - "-S", "mcu/project"
  
    > -S = 源代码目录（source）告诉 CMake：我的 CMakeLists.txt 放在 `mcu/project/ ` 这个文件夹下。

  - "-B", "mcu/build"
  
    > -B = 构建目录（build）。CMake 会把生成的 Ninja 等文件全部放进 `mcu/build/` 。

- "options": { "cwd": "${workspaceFolder}" }"
  
  > 告诉 VS Code：在运行命令前，把工作目录切换到当前项目根目录。你运行 cmake 的起点就是 ${workspaceFolder}。否则相对路径可能会错。

### CMake Build

这是第二个：

```json
{
  "label": "CMake Build",
  "type": "shell",
  "command": "cmake",
  "args": [
    "--build",
    "${cwd}/mcu/build",
    "--target",
    "all"
  ],
  "options": {
    "cwd": "${workspaceFolder}"
  }
}
```

这段 task 用来真正执行编译。前一个 `CMake Configure` 负责生成构建系统，这一个负责调用 CMake 去执行构建目录中的 Ninja 规则。

其中

- "label": "Cmake Build"
  > VS Code 中显示的任务名。当前 `tasks.json` 里实际写的是 `Cmake Build`，大小写和前面的示例略有不同，但本质上只是名字。

- "type": "shell"
  > 仍然表示通过 shell 调用外部命令。

- "command": "cmake"
  > 这里依然是调用 CMake，不过这次不是做配置，而是通过 `--build` 子命令触发构建。

- "args": [...]
  > 传给 `cmake` 的构建参数。

  - "--build"

    > 告诉 CMake：进入构建模式，而不是重新配置工程。

  - "${workspaceFolder}/build"

    > 指定构建目录。CMake 会去这个目录中寻找前面 `Configure` 生成的 Ninja 文件，并执行真正的编译流程。

  - "--target"

    > 指定要构建的目标（target）。

  - "all"

    > 表示构建默认的全部目标。通常就是把当前工程最终需要生成的产物都编出来。

- "options": { "cwd": "${workspaceFolder}" }

  > 运行前先切到工程根目录，保证相对路径解析稳定。

### Flash

这是第三个：

```json
{
    "label": "Flash",
    "type": "shell",
    "command": "openocd",
    "args": [
        "-f", "interface/cmsis-dap.cfg",
        "-f", "target/stm32f4x.cfg",
        "-c",
        "program build/template.elf verify reset exit"
    ],
    "options": {
        "cwd": "${workspaceFolder}"
    },
    "dependsOn": "Build",
    "problemMatcher": []
}
```

这段 task 用来把已经编译好的固件烧录到 STM32 芯片中。它通过 `openocd` 连接调试器，然后执行下载、校验和复位。

其中

- "label": "Flash"
  > 任务名称，表示“烧录固件”。

- "type": "shell"
  > 仍然是通过 shell 执行命令。

- "command": "openocd"
  > 调用 OpenOCD。它是嵌入式开发里常见的下载和调试工具，可以连接 CMSIS-DAP、ST-Link 等调试器。

- "args": [...]
  > 这是传给 `openocd` 的启动参数。

  - "-f", "interface/cmsis-dap.cfg"

    > 加载调试器接口配置文件。这里指定使用 CMSIS-DAP 类型的下载器。

  - "-f", "target/stm32f4x.cfg"

    > 加载目标芯片配置文件。这里指定目标是 STM32F4 系列。

  - "-c"

    > 告诉 OpenOCD：后面跟的是一段命令字符串。

  - "program build/template.elf verify reset exit"

    > 这是实际执行的 OpenOCD 命令，含义如下：
    >
    > `program build/template.elf`：把 `build/template.elf` 下载到芯片。
    >
    > `verify`：下载后做一次校验，确认写入正确。
    >
    > `reset`：烧录完成后复位芯片，让新程序开始运行。
    >
    > `exit`：执行完成后退出 OpenOCD。

- "options": { "cwd": "${workspaceFolder}" }

  > 在工程根目录执行，保证 `build/template.elf` 这个相对路径能被正确找到。

- "dependsOn": "Build"

  > 表示这个任务依赖另一个任务，VS Code 会先执行被依赖的任务，再执行当前任务。
  >
  > 不过这里要注意：当前 `tasks.json` 中实际定义的任务名是 `Cmake Build`，而这里写的是 `Build`。如果保持现状，可能会导致依赖找不到。更稳妥的写法应该让它和真实的任务 `label` 完全一致。

- "problemMatcher": []

  > 这里显式写空数组，表示不使用 VS Code 的错误输出匹配器。因为烧录任务通常不是编译器输出，不需要去解析成“代码错误”。

### Clean

这是第四个：

```json
{
  "label": "Clean",
  "type": "shell",
  "windows": {
    "command": "powershell",
    "args": [
      "-Command",
      "if (Test-Path ${workspaceFolder}/build) {Remove-Item ${workspaceFolder}/build -Recurse -Force}; if (Test-Path ${workspaceFolder}/.cache) {Remove-Item ${workspaceFolder}/.cache -Recurse -Force}"
    ]
  },
  "linux": {
    "command": "bash",
    "args": ["-c", "rm -rf ${workspaceFolder}/build ${workspaceFolder}/.cache || true"]
  },
  "osx": {
    "command": "bash",
    "args": ["-c", "rm -rf ${workspaceFolder}/build ${workspaceFolder}/.cache || true"]
  },
  "presentation": {
    "reveal": "always",
    "panel": "shared"
  },
  "problemMatcher": [],
  "options": {
    "cwd": "${workspaceFolder}"
  }
}
```

这段 task 用来清理构建产生的临时目录，把工程恢复到“未构建”的状态。由于不同操作系统的命令不同，所以它分别写了 Windows、Linux 和 macOS 三套执行方式。

其中

- "label": "Clean"
  > 任务名称，表示“清理构建目录”。

- "type": "shell"
  > 使用 shell 执行系统命令。

- "windows": { ... }
  > 当你在 Windows 上运行这个 task 时，VS Code 会优先使用这里面的命令，而不是顶层的 `command`。

  - "command": "powershell"

    > 使用 PowerShell 执行清理命令。

  - "args": [ "-Command", "..." ]

    > `-Command` 表示后面直接跟一段 PowerShell 命令。
    >
    > 这段命令的逻辑是：如果 `build` 目录存在就删除它；如果 `.cache` 目录存在也删除它。

- "linux": { ... }
  > 当在 Linux 上执行时，使用这里定义的命令。

  - "command": "bash"

    > 调用 Bash。

  - "args": ["-c", "rm -rf ${workspaceFolder}/build ${workspaceFolder}/.cache || true"]

    > `-c` 表示后面是一整段 shell 命令。
    >
    > `rm -rf ...` 会强制递归删除 `build` 和 `.cache`。
    >
    > `|| true` 的作用是：即使删除失败，整个 task 也不要因为这个命令返回非零而中断。

- "osx": { ... }
  > macOS 下与 Linux 一样，也使用 Bash 和同一套清理命令。

- "presentation": { ... }
  > 控制 VS Code 终端里这个任务的显示方式。

  - "reveal": "always"

    > 每次运行任务时，都自动把终端面板弹出来，方便你看到执行结果。

  - "panel": "shared"

    > 复用同一个终端面板，而不是每次新开一个终端窗口。

- "problemMatcher": []

  > 仍然不做错误匹配。因为这是文件清理操作，不需要把输出解析成编译错误。

- "options": { "cwd": "${workspaceFolder}" }

  > 运行前切换到项目根目录，保证 `${workspaceFolder}` 对应的是当前工程。
