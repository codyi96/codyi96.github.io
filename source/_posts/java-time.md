---
title: Java 时间与日期
catalog: true
date: 2020-09-02 21:25:38
subtitle:
header-img: header.jpg
tags:
- java
- TimeZone
---

<img src="world_time_zones.png">

## 基本概念

### 时区（TimeZone）
[时区](https://en.wikipedia.org/wiki/Time_zone)一般指理论时区，它以能被`15整除`的`经线`为中心，向东西两侧延伸7.5°，即每15°一个时区。时区的时间采用其中央经线的地方时，相邻时区时差为一个小时，共`24个时区`。其中，以本初子午线（经度0°）为中央经线的时区为零时区。

### 格林尼治标准时间(GMT)
[格林尼治标准时间(GMT, 旧称格林尼治平均时间)](https://en.wikipedia.org/wiki/Greenwich_Mean_Time)的正午是指当太阳横穿格林尼治子午线时（也就是在格林尼治上空最高点时）的时间。由于地球🌏每天的自转并不是匀速的，因此GMT实际上是有细微误差的，正因为这样，它已经逐渐被[协调世界时(UTC)](https://en.wikipedia.org/wiki/Coordinated_Universal_Time)取代。

### 协调世界时(UTC)
[协调世界时(UTC)](https://en.wikipedia.org/wiki/Coordinated_Universal_Time)又称作世界协调时间、世界协调时。它基于[国际原子时](https://en.wikipedia.org/wiki/International_Atomic_Time)，通过不规则地加入闰秒来抵消地球自转变慢带来的影响。
UTC 与 GMT 的差值极小，在日常生活中可以互换，但在高精度的科学研究中，GMT 已不再被认同。

> 北京时间(CST, 又称中国标准时间) 为 UTC +8

### Unix时间戳(Unix timestamp)
[Unix时间戳](https://en.wikipedia.org/wiki/Unix_time)也作Unix时间(Unix time)、POSIX时间(POSIX time)等，是Unix或类Unix系统使用的时间表示方式，其值为`UTC 1970-01-01 00:00:00至今的总秒数`，不考虑闰秒。

## JDK8 以前

### Date
[Date](https://docs.oracle.com/javase/8/docs/api/java/util/Date.html)是第一代时间类，自 JDK1.0 引入。
Date 本身没有时区的概念，因此不支持配置时区，它的`toString()`、`toLocaleString()`、`toGMTString()`方法使用默认的`Locale`、`TimeZone`对当前时间做解析:
```java
Date date = new Date();
System.out.println("当前时间: " + date.getTime()); // 1599061157549
System.out.println("String: " + date.toString()); // Wed Sep 02 23:39:17 CST 2020
System.out.println("Locale String: " + date.toLocaleString()); // 2020-9-2 23:39:17
System.out.println("GMT String: " + date.toGMTString()); // 2 Sep 2020 15:39:17 GMT
```

### Calendar
[Calendar](https://docs.oracle.com/javase/7/docs/api/java/util/Calendar.html)是`Date`的升级版，自 JDK1.1 引入。
相比于`Date`，`Calendar`主要有两大优势:
- 提供了更丰富的日期运算方法
- 支持配置时区

```java
Calendar calendar = Calendar.getInstance();
System.out.println("本地时间:" + calendar.get(Calendar.HOUR_OF_DAY) + "时"); // 本地时间:0时
calendar.setTimeZone(TimeZone.getTimeZone("GMT+1"));
System.out.println("东1区时间:" + calendar.get(Calendar.HOUR_OF_DAY) + "时"); // 东1区时间:17时
```

### DateFormat
[DateFormat](https://docs.oracle.com/javase/7/docs/api/java/text/DateFormat.html)用来做时间和字符串之间的转换，可以通过`DateFormat.getDateInstance()`、`DateFormat.getTimeInstance()`等方式获取实例。一般情况下，我们更倾向于直接使用[SimpleDateFormat](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html):

```java
String pattern = "GGG yyyy MMMMM dd EEE hh:mm aaa";
Date date = new Date();
DateFormat df = new SimpleDateFormat(pattern, Locale.SIMPLIFIED_CHINESE);
DateFormat dfEn = new SimpleDateFormat(pattern, Locale.US);
// 当前时间:公元 2020 九月 03 星期四 12:43 上午
System.out.println("当前时间:" + df.format(date));
// 当前时间(美式表达):AD 2020 September 03 Thu 12:43 AM
System.out.println("当前时间(美式表达):" + dfEn.format(date));
df.setTimeZone(TimeZone.getTimeZone("GMT+1"));
// 东1区时间:公元 2020 九月 02 星期三 05:43 下午
System.out.println("东1区时间:" + df.format(date));
```

`DateFormat`支持配置`Locale`和`TimeZone`:

- Locale: 设置表达格式。上面的例子中，中式表达和美式表达在部分字段有区别。
- TimeZone: 设置时区。

👿 使用`DateFormat`时，最好使用严格的日期解析模式:
```java
DateFormat.setLenient(false); // 宽松模式，默认为true
```

😞Bad Code
```java
String pattern = "yyyy-MM-dd";
DateFormat df = new SimpleDateFormat(pattern, Locale.SIMPLIFIED_CHINESE);
Date date = df.parse("2020-02-30");
// 时间:2020-03-01
System.out.println("时间:" + df.format(date));
```
外部传入错误的`2020-02-30`，自动解析为`2020-03-01`

😘Good Code
```java
String pattern = "yyyy-MM-dd";
DateFormat df = new SimpleDateFormat(pattern, Locale.SIMPLIFIED_CHINESE);
df.setLenient(false); // key code
Date date = df.parse("2020-02-30");
System.out.println("时间:" + df.format(date));
```
外部传入错误的`2020-02-30`，代码执行报错:

```log
Exception in thread "main" java.text.ParseException: Unparseable date: "2020-02-30"
	at java.text.DateFormat.parse(DateFormat.java:366)
	at com.codyi.Main.main(Main.java:17)
```
这样一来，只要在代码中及时捕获异常，就能准确拦截错误的输入，保证程序正常运行。

## JDK8

在 JDK8 中，引入了 LocalDate、LocalTime、LocalDateTime、Instant、Clock 等新的时间类，由于目前还未深入使用，暂时不做说明。 :D
