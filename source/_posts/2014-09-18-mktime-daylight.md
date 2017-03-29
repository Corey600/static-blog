---
layout: post
title: 标准库函数mktime与夏令时
category : 备忘
tagline: "备忘"
tags : [C++,夏令时]
excerpt_separator: <!--more-->
---

#### 1.现象描述

将本地系统时区设置为“(UTC)都柏林，爱丁堡，里斯本，伦敦”，并开启自动夏令时，将系统时间设置为处于当地的夏令时时间段内。代码中每次对时间进行转换，结果都会偏移一个小时。

#### 2.现象分析

定位代码到C标准库time.h的函数mktime，struct tm结构的年月日的表示时间通过mktime转换为时间戳之后会偏移一个小时。原因是夏令时标志位tm_isdst没有设置，默认为0，表示不是夏令时。系统在调用mktime时会判断当前时区是否启用夏令时，如果启用夏令时，而传入的时间事实上处于夏令时，但是夏令时标志为0，系统就会加一个小时做默认的转换，并将夏令时标志置为1。

#### 3.解决问题

<!--more-->

首先明确什么是夏令时。维基百科的说明如下：

>夏时制或夏令时间（英语：Summer time），又称日光节约时制、日光节约时间（英语：Daylight saving time），是一种为节约能源而人为规定地方时间的制度，在这一制度实行期间所采用的统一时间称为“夏令时间”。一般在天亮早的夏季人为将时间提前一小时，可以使人早起早睡，减少照明量，以充分利用光照资源，从而节约照明用电。各个采纳夏时制的国家具体规定不同。

简单的说，某些国家会将一年中的某一段时间调快一个小时。那么在本机系统开启夏令时的情况下，调用mktime就有以下几种情况：

1. 传入的时间根据当地的夏令时规定正处于实行夏令时的时间段，且标志位为1，调用mktime不进行偏移。
2. 传入的时间根据当地的夏令时规定正处于实行夏令时的时间段，但标志位为0，调用mktime加一个小时，并置标志位为1。
3. 传入的时间根据当地的夏令时规定正处于实行夏令时的时间段，但标志位为-1，调用mktime不进行偏移，并置标志位为1。
4. 传入的时间根据当地的夏令时规定不处于实行夏令时的时间段，但标志位为1，调用mktime减一个小时，并置标志位为0。
5. 传入的时间根据当地的夏令时规定不处于实行夏令时的时间段，但标志位为0，调用mktime不进行偏移，并置标志位为0。
6. 传入的时间根据当地的夏令时规定不处于实行夏令时的时间段，但标志位为-1，调用mktime不进行偏移，并置标志位为0。

可以发现，如果置夏令时标志为-1，实质上是让系统自己判断传入的时间是否是夏令时。

代码示例

传入时间处于非夏令时时间段：

```
struct tm stTime={0};
time_t tTime;
stTime.tm_year = 2014;
stTime.tm_mon = 12;
stTime.tm_mday = 26;
stTime.tm_hour = 1;
stTime.tm_min = 30;
stTime.tm_sec = 12;
stTime.tm_mon -= 1;
stTime.tm_year -= 1900;

stTime.tm_isdst = -1; // 设置夏令时标志为-1，表示该时间是否是夏令时未知
printf("\n++++++++++[标记stTime.tm_isdst = %d]++++++++++\n\n", stTime.tm_isdst);
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
tTime = mktime(&stTime);
printf("\n运行tTime = mktime(&stTime);语句之后\n\n");
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
    
stTime.tm_isdst = 0; // 设置夏令时标志为0，表示该时间不是夏令时
printf("\n++++++++++[标记stTime.tm_isdst = %d]++++++++++\n\n", stTime.tm_isdst);
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
tTime = mktime(&stTime);
printf("\n运行tTime = mktime(&stTime);语句之后\n\n");
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
    
stTime.tm_isdst = 1; // 设置夏令时标志为1，表示该时间是夏令时
printf("\n++++++++++[标记stTime.tm_isdst = %d]++++++++++\n\n", stTime.tm_isdst);
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
tTime = mktime(&stTime);
printf("\n运行tTime = mktime(&stTime);语句之后\n\n");
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
```

运行结果：

```
++++++++++[标记stTime.tm_isdst = -1]++++++++++
stTime时间：20141226T013012Z 夏令时标志：-1
运行tTime = mktime(&stTime);语句之后
stTime时间：20141226T013012Z 夏令时标志：0
++++++++++[标记stTime.tm_isdst = 0]++++++++++
stTime时间：20141226T013012Z 夏令时标志：0
运行tTime = mktime(&stTime);语句之后
stTime时间：20141226T013012Z 夏令时标志：0
++++++++++[标记stTime.tm_isdst = 1]++++++++++
stTime时间：20141226T013012Z 夏令时标志：1
运行tTime = mktime(&stTime);语句之后
stTime时间：20141226T003012Z 夏令时标志：0
```

传入时间处于夏令时时间段：

```
struct tm stTime={0};
time_t tTime;
stTime.tm_year = 2014;
stTime.tm_mon = 8;
stTime.tm_mday = 26;
stTime.tm_hour = 1;
stTime.tm_min = 30;
stTime.tm_sec = 12;
stTime.tm_mon -= 1;
stTime.tm_year -= 1900;

stTime.tm_isdst = -1; // 设置夏令时标志为-1，表示该时间是否是夏令时未知
printf("\n++++++++++[标记stTime.tm_isdst = %d]++++++++++\n\n", stTime.tm_isdst);
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
tTime = mktime(&stTime);
printf("\n运行tTime = mktime(&stTime);语句之后\n\n");
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);

stTime.tm_isdst = 1; // 设置夏令时标志为1，表示该时间是夏令时
printf("\n++++++++++[标记stTime.tm_isdst = %d]++++++++++\n\n", stTime.tm_isdst);
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
tTime = mktime(&stTime);
printf("\n运行tTime = mktime(&stTime);语句之后\n\n");
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);

stTime.tm_isdst = 0; // 设置夏令时标志为0，表示该时间不是夏令时
printf("\n++++++++++[标记stTime.tm_isdst = %d]++++++++++\n\n", stTime.tm_isdst);
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
tTime = mktime(&stTime);
printf("\n运行tTime = mktime(&stTime);语句之后\n\n");
printf("stTime时间：%04d%02d%02dT%02d%02d%02dZ 夏令时标志：%d\n",
    stTime.tm_year+1900, stTime.tm_mon+1, stTime.tm_mday,  stTime.tm_hour,  stTime.tm_min,  stTime.tm_sec, stTime.tm_isdst);
```

运行结果：

```
++++++++++[标记stTime.tm_isdst = -1]++++++++++
stTime时间：20140826T013012Z 夏令时标志：-1
运行tTime = mktime(&stTime);语句之后
stTime时间：20140826T013012Z 夏令时标志：1
++++++++++[标记stTime.tm_isdst = 1]++++++++++
stTime时间：20140826T013012Z 夏令时标志：1
运行tTime = mktime(&stTime);语句之后
stTime时间：20140826T013012Z 夏令时标志：1
++++++++++[标记stTime.tm_isdst = 0]++++++++++
stTime时间：20140826T013012Z 夏令时标志：0
运行tTime = mktime(&stTime);语句之后
stTime时间：20140826T023012Z 夏令时标志：1
```

#### 4.总结

mktime对时间进行偏移的原理已经清楚了，那么代码事实上只是要保证无论在什么时区下，是否开启夏令时，时间的转换都不能出现偏移。那就置夏令时标志位为-1，系统会自动判断传入的时间是否是夏令时，并相应地标志夏令时标志位。

#### 5.相关延伸—时间转换对时区的考虑

MSDN里面对mktime的说明有下面这么一句话：

> After an adjustment to UTC, _mktime32 handles dates from midnight, January 1, 1970, to 03:14:07, January 19, 2038. _mktime64 handles dates from midnight, January 1, 1970 to 23:59:59, December 31, 3000. This adjustment may cause these functions to return -1 (cast to time_t, __time32_t or __time64_t) even though the date you specify is within range. For example, if you are in Cairo, Egypt, which is two hours ahead of UTC, two hours will first be subtracted from the date you specify in timeptr; this may now put your date out of range.

意思大概说的是mktime把时间转换为时间戳并调整为UTC标准时间以后的时间范围。而后面一句说，这种调整为UTC标准的做法会引起函数返回-1，也就是说超出范围。比如你在埃及开罗，它比标准时间早两个小时，即使你在埃及时间是在“midnight, January 1, 1970, to 03:14:07, January 19, 2038”范围内，调用mktime，将转换为UTC时间，也就是加上两个小时就可能超出范围，函数返回-1。这就变相的在说明，mktime的时间转换是会默认认为你传入的时间是本地时间，而基准1970年1月1日则是UTC标准时间，那就先要将你传入的时间根据本机时区转换成UTC时间，再计算时间戳。也就是说在不同的本机系统时区下面，传入相同的年月日几分几秒得到的时间戳是不同的。
 