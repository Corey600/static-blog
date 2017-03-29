---
layout: post
title: Ubuntu下SublimeText2的G++和Python配置
category : 备忘
tagline: "备忘"
tags : [Ubuntu, SublimeText2, G++, Python]
excerpt_separator: <!--more-->
---

如果原本系统已经安装了g++和python2.7，并且配置好了环境变量，那么sublime本身默认的配置文件已经能够满足要求了。但是对于我这个用惯了codeblock的人来说，运行时不能在终端输入是无法忍受的。那么我就希望将sublime配置成codeblock那样跳出终端窗口来显示输出，并且能够输入。不多说，先贴出配置文件。

g++配置:

    {
        "cmd": ["g++", "${file}", "-o", "${file_path}/${file_base_name}"],
        "file_regex": "^(..[^:]*):([0-9]+):?([0-9]+)?:? (.*)$",
        "working_dir": "${file_path}",
        "selector": "source.c, source.c++",

        "variants":
        [
            {
                "name": "Run",
                "cmd":["x-terminal-emulator", "-x", "bash", "-c", "g++ '${file}' -o '${file_path}/${file_base_name}' && '${file_path}/${file_base_name}' ;read -n1 -p 'press any key to continue.'"]
            }
        ]
    }
    
<!--more-->

python配置:

    {
        "cmd":["x-terminal-emulator", "-x", "bash", "-c", "python -u '${file}' ;read -n1 -p 'press any key to continue.'"],
        //"cmd": ["python", "-u", "$file"],
        "file_regex": "^[ ]*File \"(...*?)\", line ([0-9]*)",
        "selector": "source.python"
    }

说明：
其中命令``x-terminal-emulator -x``是实现了跳出新的窗口，由于我用的是linux-mint，默认桌面环境是mate，如果是gnome那么命令可能应该是``gnome-terminal -x``。

而命令``;read -n1 -p ”press any key to continue.“``则是为了实现窗口在输出完毕后不立即关闭，而是再显示press any key to continue.并由你按下enter时窗口关闭。这里的any key其实并不是任意键，目前测试只接受英文字符和enter。

其他命令就不多说了，可以参考sublime的原有配置或者参考官网。
