---
layout: post
title: 完善TQ2440上Linux-2.6.32.59的串口驱动
category : 备忘
tagline: "备忘"
tags : [TQ2440, Linux, 串口]
excerpt_separator: <!--more-->
---

修改内核源码``arch/arm/mach-s3c2440/mach-smdk2440.c``文件的100行，将其改为：

    .ulcon =0x03,

修改``drivers/serial/samsung.c``文件的53行添加如下内容：

    #include <mach/regs-gpio.h>

然后在397行左右，函数``static int s3c24xx_serial_startup(struct uart_port *port)``之前添加

<!--more-->

    extern void s3c2410_gpio_cfgpin(unsigned int pin, unsigned int function);

    extern void s3c2410_gpio_pullup(unsigned int pin, unsigned int to);

然后在462行左右添加以下注释说明处的代码：

    dbg("s3c24xx_serial_startup ok\n");
    //添加以下代码
    if (port->line == 2) {
        s3c2410_gpio_cfgpin(S3C2410_GPIONO(S3C2410_GPIO_BANKH, 6), S3C2410_GPH6_TXD2);
        s3c2410_gpio_pullup(S3C2410_GPIONO(S3C2410_GPIO_BANKH, 6), 1);
        s3c2410_gpio_cfgpin(S3C2410_GPIONO(S3C2410_GPIO_BANKH, 7), S3C2410_GPH7_RXD2);
        s3c2410_gpio_pullup(S3C2410_GPIONO(S3C2410_GPIO_BANKH, 7), 1);
    }
    //添加以上代码
    return ret;
     err:
    s3c24xx_serial_shutdown(port);
    return ret;

下面列出串口配置情况：

    Device Drivers  -->
        Character devices  -->
            Serial drivers  -->
                <>8250/16550 and compatible serial support
                  ***Non-8250 serial port support***
                <*>Samsung SoC serial support
                [*]Support for console on Samsung SoC serial port
                <*>Samsung S3C2440/S3C2442 Serial port support
