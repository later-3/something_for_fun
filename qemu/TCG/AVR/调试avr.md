qemu avr的作者已经写了一个测试avr的小程序 [seharris/qemu-avr-tests: Tests produced while working on AVR emulation for QEMU (github.com)](https://github.com/seharris/qemu-avr-tests)

我们将它跑起来。

```shell
sudo apt-get install gcc-avr
sudo apt-get install gdb-avr
sudo apt-get install arduino

bash build.sh
```

bin下面生成了一些elf文件，调试第一个。
`launch.json`
```json
{
    "version": "0.2.0",
    "configurations": [
      {
        "name": "C/C++: g++ build and debug active file",
        "type": "cppdbg",
        "request": "launch",
        "program": "/home/later/code/qemu/build/qemu-system-avr",
        "stopAtEntry": false,
        "cwd": "${workspaceFolder}",
        "environment": [],
        "externalConsole": false,
        "MIMode": "gdb",
        "miDebuggerPath": "/usr/bin/gdb",
        "args": [
            "-nographic",
            "-M", "mega2560",
            "-bios", "/home/later/code/qemu-avr-tests/instruction-tests/bin/BA.elf",
            "-s", "-S"
        ],
        "setupCommands": [
          {
            "description": "Enable pretty-printing for gdb",
            "text": "-enable-pretty-printing",
            "ignoreFailures": true
          }
        ],
      }
    ]
  }
```
如果之前为了看看avr的编译和代码结构，有复制avr的目录然后重命名，可能会重复注册qom，在meson.build中取消掉。

```gdb
avr-gdb bin/BA.elf
target remote:1234
layout asm
n
```

![](https://raw.githubusercontent.com/later-3/img_picgo/main/img/20230506224935.png)
