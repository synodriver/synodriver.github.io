---
date: '2025-03-13T16:53:32+08:00'
draft: false
title: '在 CLion 中使用 ESP-IDF'
author: 'synodriver'
tags: ["ESP32-S3", "ESP-IDF", "CLion", "IDE", "教程"]
---

# 在 CLion 中使用 ESP-IDF

本教程介绍如何在跨平台 C/C++ IDE CLion 中使用 ESP-IDF 项目。我们将构建一个应用程序，将其烧录到 ESP32-S3 开发板并进行调试，还会运行串口监视器，并查看用于自定义 ESP-IDF 项目的配置菜单。所有操作都将在 IDE 内完成，无需切换到系统终端或其他工具。

本教程面向初学者，即使你从未使用过 ESP-IDF 和 CLion，也可以跟随操作。

## 前置条件

我们将使用以下硬件：

- ESP32-S3-DevKitC-1 v1.1。
- MacBook Pro M2 Max。
- USB-C 转 micro USB 数据线。

开始之前，请完成以下准备工作：

- [安装 ESP-IDF 工具链](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/get-started/linux-macos-setup.html#standard-toolchain-setup-for-linux-and-macos)。
- [安装 CLion](https://www.jetbrains.com/clion/download/)。你可以使用免费的 30 天试用版，也请确认是否有[折扣或免费选项](https://www.jetbrains.com/clion/buy/?section=discounts&billing=yearly)可用。

虽然本文在 macOS 上运行 CLion，但除本文特别说明的情况外，Windows 和 Linux 上的工作流程与设置基本相同。如果你想进一步了解本教程未涉及的 CLion 配置选项，请参阅 [CLion 文档](https://www.jetbrains.com/help/clion/installation-guide.html)。

## 配置 ESP-IDF 项目

1. 启动 CLion。
2. 在欢迎界面选择 `Open`。

![在 CLion 中打开项目](https://developer.espressif.com/blog/clion/img/1-esp-clion-open-project.webp)

3. 进入计算机上的默认 ESP-IDF 目录。本教程中该目录为 `/Users/username/esp/esp-idf`。然后进入 `examples` 子目录，选择要构建的项目。

本教程使用 `led_strip_simple_encoder`，它位于 `examples/peripherals/rmt`。该应用会在开发板上生成 LED 彩虹追逐效果。虽然它原本用于 LED 灯带，但也适用于本文使用的单颗 LED 开发板。LED 会按预定顺序闪烁不同颜色。

4. 点击 `Trust Project`，随后会打开 `Open Project Wizard`。
5. 点击 `Manage toolchains...`。

![管理工具链](https://developer.espressif.com/blog/clion/img/2-esp-clion-manage-toolchains.webp)

6. 点击 `+`，选择 `System`（Windows 用户请选择 `MinGW`），创建新的工具链。名称可以自定义。

   - 在新的工具链模板中选择 `Add environment` > `From file`。

   ![从文件添加环境](https://developer.espressif.com/blog/clion/img/3-esp-clion-add-environment.webp)

   - 点击 `Browse...`。

   ![浏览环境文件](https://developer.espressif.com/blog/clion/img/3.1-esp-clion-add-environment-browse.webp)

   - 选择计算机上的环境文件。在 macOS 上，该文件名为 `export.sh`；在 Windows 上为 `export.bat`。它位于默认 ESP-IDF 目录中。
   - 点击 `Apply`。

7. 进入 `Settings` > `Build, Execution, Deployment` > `CMake`。

   - 在默认的 `Debug` 配置中，选择刚创建的工具链，本例中为 `ESP-IDF`。

   ![配置 CMake 工具链](https://developer.espressif.com/blog/clion/img/4-esp-clion-cmake.webp)

   - 在 `CMake options` 字段中输入 `-DIDF_TARGET=esp32s3`（因为使用的是基于 ESP32-S3 的开发板）。
   - 在 `Build directory` 字段中输入 `build`。
   - 点击 `OK`。

项目随后会开始加载。如果加载失败，请在 CMake 工具窗口的设置中点击 `Reset Cache and Reload Project`。

![重新加载项目](https://developer.espressif.com/blog/clion/img/5-esp-clion-reload-project.webp)

如果项目加载成功，你会在 CMake 日志末尾看到 `[Finished]`。现在可以构建应用并将其烧录到开发板了。

## 构建应用并烧录开发板

1. 确保开发板通过 UART 端口连接到计算机。
2. 如果使用相同的示例应用，请确认源代码中正确设置了 GPIO LED 编号：

   - 在 CLion 的 `Project` 工具窗口中，找到项目目录下的 `main` 目录，并打开 `led_strip_example_main.c` 文件。
   - 在 `#define RMT_LED_STRIP_GPIO_NUM` 行中，根据开发板硬件版本，将默认值改为 `38` 或 `48`。

   ![设置 GPIO LED 编号](https://developer.espressif.com/blog/clion/img/6-esp-clion-gpio-num.webp)

3. 点击主工具栏中的 `Run / Debug Configurations` 下拉列表，选择 `flash` 配置。该配置会先构建项目，然后自动烧录开发板。

![选择 flash 配置](https://developer.espressif.com/blog/clion/img/7-esp-clion-flash-target.webp)

4. 点击 IDE 主工具栏上的绿色 `Build` 图标。

![构建并烧录](https://developer.espressif.com/blog/clion/img/8-esp-clion-build-flash.webp)

在 `Messages` 工具窗口中，可以查看构建和烧录过程的信息。

![构建和烧录完成](https://developer.espressif.com/blog/clion/img/9-esp-clion-build-flash-finished.webp)

构建完成后，开发板上的 LED 会按照配置的彩虹追逐模式闪烁。

![开发板上的彩虹追逐效果](https://developer.espressif.com/blog/clion/img/10-esp-clion-build-flash-board.webp)

要更改追逐速度，请修改 `led_strip_example_main.c` 中的 `EXAMPLE_ANGLE_INC_FRAME` 值。要更改颜色密度，请修改同一文件中的 `EXAMPLE_ANGLE_INC_LED`。

## 运行 IDF 监视器

1. 从工具链设置中复制环境文件的路径。本教程中的路径为 `/Users/Oleg.Zinovyev/esp/esp-idf/export.sh`。
2. 进入 `Run | Edit Configurations`，点击 `Add New Configuration`。

![添加运行配置](https://developer.espressif.com/blog/clion/img/11-esp-clion-add-config.webp)

3. 选择 `Shell Script` 模板。在新的配置对话框中：

   - 输入自定义名称。
   - 在 `Execute` 旁边选择 `Script text`。
   - 输入以下文本，其中包含刚才复制的环境文件路径：`. /Users/Oleg.Zinovyev/esp/esp-idf/export.sh ; idf.py flash monitor`。

   ![配置 flash monitor](https://developer.espressif.com/blog/clion/img/12-esp-clion-flash-monitor.webp)

   - 其余选项保持不变，点击 `OK`。

4. 点击主工具栏上的绿色 `Run` 图标。

随后，监视器输出的诊断信息会显示在 IDE 的终端中。

![IDF 监视器输出](https://developer.espressif.com/blog/clion/img/13-esp-clion-monitor.webp)

## 使用项目配置菜单

项目配置菜单是在终端中运行的图形界面工具，可用于配置 ESP-IDF 项目。它基于 Kconfig，提供多种底层配置选项，包括启动加载程序、串行闪存和安全功能的配置。

项目配置菜单通过 `idf.py menuconfig` 命令运行，因此需要相应地配置运行配置。

1. 打开之前创建的、用于运行串口监视器的配置。
2. 点击 `Copy Configuration`。

![复制配置](https://developer.espressif.com/blog/clion/img/14-esp-clion-copy-config.webp)

3. 将复制出的配置重命名，以体现其新功能，例如 `ESP-menu-config`。
4. 在脚本文本中，将 `flash monitor` 替换为 `menuconfig`。

![配置 menuconfig](https://developer.espressif.com/blog/clion/img/15-esp-clion-menu-config.webp)

5. 点击 `OK`。
6. 确保禁用 IDE 的新终端选项（取消勾选），否则项目配置菜单可能无法正常工作。

![禁用 IDE 新终端](https://developer.espressif.com/blog/clion/img/17-esp-clion-menu-config-terminal.webp)

7. 点击主工具栏上的绿色 `Run` 图标。

项目配置菜单会在 IDE 的终端中打开。

![运行项目配置菜单](https://developer.espressif.com/blog/clion/img/16-esp-clion-menu-config-run.webp)

你可以使用键盘浏览菜单并修改项目的默认参数，例如闪存大小。

![修改闪存大小](https://developer.espressif.com/blog/clion/img/18-esp-clion-menu-config-flash-size.webp)

## 在终端中使用 `idf.py` 命令

你还可以在终端中使用带有不同选项的 `idf.py` 命令来管理项目并查看其配置。例如，下面是 `idf.py size` 命令输出的固件大小信息：

![idf.py size 输出](https://developer.espressif.com/blog/clion/img/19-esp-clion-idf-py.webp)

你可以将经常使用的命令配置为 Shell 脚本，并将它们作为独立配置运行，就像前面访问串口监视器和项目配置菜单时所做的那样。

要详细了解 `idf.py` 的选项，请阅读[官方文档](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/build-system.html#idf-py)。

## 调试项目

我们将使用 `Debug Servers` 配置选项调试项目。CLion 的这一功能可以方便地为不同构建目标配置和使用调试服务器。

1. 从开发板的 UART 接口拔下 USB 数据线，然后将其插入 USB 接口。
2. 确保在 `Settings` > `Advanced Settings` > `Debugger` 中启用了 `Debug Servers`。

![启用调试服务器](https://developer.espressif.com/blog/clion/img/20-esp-clion-enable-debug-servers.webp)

3. 在主工具栏的切换器中选择 `led_strip_simple_encoder.elf` 配置。

![选择 ELF 配置](https://developer.espressif.com/blog/clion/img/21-esp-clion-led-strip-config.webp)

随后，主工具栏中会出现 `Debug Servers` 切换器。

4. 选择 `Edit Debug Servers`。

![编辑调试服务器](https://developer.espressif.com/blog/clion/img/22-esp-clion-edit-debug-server.webp)

5. 点击 `+` 添加新的调试服务器。
6. 选择 `Generic` 模板。

![选择 Generic 模板](https://developer.espressif.com/blog/clion/img/23-esp-clion-generic-template.webp)

7. 在这里需要指定几个参数，其中一部分取决于你的开发板。本教程使用以下设置：

   - `GDB Server` > `Executable`：`/Users/Oleg.Zinovyev/.espressif/tools/openocd-esp32/v0.12.0-esp32-20241016/openocd-esp32/bin/openocd`
   - `GDB Server` > `Arguments`：`-f board/esp32s3-builtin.cfg`

   ![配置 GDB 服务器选项](https://developer.espressif.com/blog/clion/img/24-esp-clion-gdb-server-options.webp)

   - `Device Settings` 如下：

   ![设备设置](https://developer.espressif.com/blog/clion/img/25-esp-clion-device-settings.webp)

   - `Debugger` > `Custom GDB Executable`：`/Users/Oleg.Zinovyev/.espressif/tools/xtensa-esp-elf-gdb/14.2_20240403/xtensa-esp-elf-gdb/bin/xtensa-esp32s3-elf-gdb`
   - `Debugger` > `Connection` > `Arguments`：`tcp::3333`

   ![调试器选项](https://developer.espressif.com/blog/clion/img/26-esp-clion-debugger-options.webp)

此外，最好在 `Debugger` 选项卡中禁用 `Persistent session`，因为该选项可能不稳定。

8. 其余默认设置保持不变，点击 `Apply`。
9. 你还可以在测试模式下运行 GDB 服务器，以验证设置是否正确。

![测试运行调试服务器](https://developer.espressif.com/blog/clion/img/27-esp-clion-debugger-test-run.webp)

测试成功时，`Test Run...` 的输出如下：

![调试服务器测试输出](https://developer.espressif.com/blog/clion/img/28-esp-clion-debugger-test-output.webp)

10. 保存更改并关闭 `Debug Servers` 配置对话框。
11. 在源代码文件中设置断点。
12. 点击主工具栏上的绿色 `Debug` 图标，启动调试会话。

之后，你可以执行所需的调试操作并查看应用程序数据。

![调试会话](https://developer.espressif.com/blog/clion/img/29-esp-clion-debugging.webp)

要进一步了解 CLion 的调试器功能，请阅读 IDE 文档。

如果需要针对特定 ESP32 芯片进行调试，请参阅制造商文档。为特定芯片配置调试服务器时，可能需要注意与 JTAG 设置相关的一些特殊情况。

## 结语

CLion 致力于成为开发各种嵌入式系统的通用、便捷工具，无论使用何种硬件、框架或工具链。ESP-IDF 也是如此：我们计划简化这类项目的工作流程，目前正在积极推进相关工作。

如果你将本教程用于 ESP-IDF 项目并向我们反馈使用体验，我们将不胜感激。如果你有任何想法或遇到问题，请通过我们的[问题跟踪器](https://youtrack.jetbrains.com/issues/CPP/)告诉我们。

> **免责声明：**
>
> 本文包含合作伙伴或社区作者提供的内容。Espressif 未对所提供的信息进行独立核实，读者应自行评估。内容的持续维护和准确性由作者负责。
