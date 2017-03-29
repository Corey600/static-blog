---
layout: post
title: Linux下用Dnw烧写bin镜像(Ubuntu for TQ2440)
category : 备忘
tagline: "备忘"
tags : [Linux, Ubuntu, TQ2440]
excerpt_separator: <!--more-->
---

我们要用到的工具是C-kermit 和 dnw2。其中C-kermit是串口连接程序。而dnw2是用来向开发板下载程序的。
首先我们安装kermit，我们可以使用``#sudo apt-get install ckermit``来安装kermit。

下载完成之后还要对其进行配置。``#sudo vim /etc/kermit/kermrc``

![Alt text](/images/20120922/1.jpg)

<!--more-->

set line /dev/ttyUSB0 (这里填写通过命令#dmesg看到的设备名。上图中红线部分。attached to ttyUSB0 说明已经附加到了ttyUSB0这个设备上了。)

    set speed 115200

    set carrier-watch off

    set handshake none

    set flow-control none

    robust

    set file type bin

    set file name lit

    set rec pack 1000

    set send pack 1000

    set window 5

配置完以后的样子

![Alt text](/images/20120922/2.jpg)

红色方框内的内容为添加内容。

使用以下命令打开kermit：``#sudo kermit``

使用以下命令链接：``>connect``

就可以使用串口了。

切换：按下Ctrl+\ ,再按c 就跳回kermit。跳回串口控制，可以输入c,即connect命令。

![Alt text](/images/20120922/3.jpg)

上图是使用kermit连接开发板。

接着说一下dnw2的使用。先把以下某大神写的代码保存成文件dnw2.c

    #include <stdio.h>
    #include <usb.h>
    #include <errno.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <unistd.h>

    #define QQ2440_SECBULK_IDVENDOR        0x5345
    #define QQ2440_SECBULK_IDPRODUCT    0x1234

    struct usb_dev_handle * open_port() {
        struct usb_bus *busses, *bus;

        usb_init();
        usb_find_busses();
        usb_find_devices();

        busses = usb_get_busses();
        for (bus = busses; bus; bus = bus->next) {
            struct usb_device *dev;
            for (dev = bus->devices; dev; dev = dev->next) {
                printf("idVendor:0x%x\t,ipProduct:0x%x\n",
                        dev->descriptor.idVendor, dev->descriptor.idProduct);
                if (QQ2440_SECBULK_IDVENDOR == dev->descriptor.idVendor
                        && QQ2440_SECBULK_IDPRODUCT == dev->descriptor.idProduct) {
                    printf("Target usb device found!\n");
                    struct usb_dev_handle *hdev = usb_open(dev);
                    if (!hdev) {
                        perror("Cannot open device");
                    } else {
                        if (0 != usb_claim_interface(hdev, 0)) {
                            perror("Cannot claim interface");
                            usb_close(hdev);
                            hdev = NULL;
                        }
                    }
                    return hdev;
                }
            }
        }

        printf("Target usb device not found!\n");

        return NULL;
    }

    void usage() {
        printf("Usage: dnw2 <file>\n\n");
    }

    unsigned char* prepare_write_buf(char *filename, unsigned int *len) {
        unsigned char *write_buf = NULL;
        struct stat fs;

        int fd = open(filename, O_RDONLY);
        if (-1 == fd) {
            perror("Cannot open file");
            return NULL;
        }
        if (-1 == fstat(fd, &fs)) {
            perror("Cannot get file size");
            goto error;
        }
        write_buf = (unsigned char*) malloc(fs.st_size + 10);
        if (NULL == write_buf) {
            perror("malloc failed");
            goto error;
        }

        if (fs.st_size != read(fd, write_buf + 8, fs.st_size)) {
            perror("Reading file failed");
            goto error;
        }

        printf("Filename : %s\n", filename);
        printf("Filesize : %d bytes\n", fs.st_size);

        *((u_int32_t*) write_buf) = 0x30000000; //download address
        *((u_int32_t*) write_buf + 1) = fs.st_size + 10; //download size;

        *len = fs.st_size + 10;
        return write_buf;
        error: if (fd != -1)
            close(fd);
        if (NULL != write_buf)
            free(write_buf);
        fs.st_size = 0;
        return NULL;
    }

    int main(int argc, char *argv[]) {
        if (2 != argc) {
            usage();
            return 1;
        }

        struct usb_dev_handle *hdev = open_port();
        if (!hdev) {
            return 1;
        }

        unsigned int len = 0;
        unsigned char* write_buf = prepare_write_buf(argv[1], &len);
        if (NULL == write_buf)
            return 1;

        unsigned int remain = len;
        unsigned int towrite;
        printf("Writing data ...\n");
        while (remain) {
            towrite = remain > 512 ? 512 : remain;
            if (towrite != usb_bulk_write(hdev, 0x03, write_buf + (len - remain),
                    towrite, 3000)) {
                perror("usb_bulk_write failed");
                break;
            }
            remain -= towrite;
            printf("\r%d%\t %d bytes     ", (len - remain) * 100 / len, len
                    - remain);
            fflush(stdout);
        }
        if (0 == remain)
            printf("Done!\n");
        return 0;
    }

然后安装 libusb 库``#sudo apt-get install libusb-dev``

编译 dnw2.c 文件``#gcc dnw2.c -o dnw2 -lusb``

把生成的可执行文件 dnw2 移动到 /opt/dnw 里面``#sudo mv dnw2 /opt/dnw``

以root权限打开``#sudo vim /etc/profile``

在文件末尾加上``export PATH=$PATH:/opt/dnw``

保存退出后运行命令``#source /etc/profile``

PC 链接 TQ2440开发板，在终端里面运行 ``#sudo kermit``  ，然后connect连接，打开串口，按相应键进入 USB 传输

再开一个终端，输入``#sudo dnw2 <文件完整路径>``

等待下载完毕。

参考链接：      
[linux环境下安装dnw（for mini2440）](http://tanglz2005.blog.163.com/blog/static/8569819620122213535490/)     
[mini2440 在 ubuntu12.04 下 minicom 串口 dnw2 实验成功](http://hi.baidu.com/lv0xian/item/dd7e26321316b880c2cf29a5)