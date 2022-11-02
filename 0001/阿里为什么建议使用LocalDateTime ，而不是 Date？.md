阿里为什么建议使用`LocalDateTime` ，而不是 Date？

> 在项目开发过程中经常遇到时间处理，但是你真的用对了吗，理解阿里巴巴开发手册中禁用static修饰`SimpleDateFormat`吗？

通过阅读本篇文章你将了解到：

- 为什么需要`LocalDate`、`LocalTime`、`LocalDateTime`【java8新提供的类】
- `java8`新的时间API的使用方式，包括创建、格式化、解析、计算、修改

为什么需要`LocalDate`、`LocalTime`、`LocalDateTime`

`Date`如果不格式化，打印出的日期可读性差

```
Tue Sep 10 09:34:04 CST 2019
```

使用`SimpleDateFormat`对时间进行格式化，但SimpleDateFormat是线程不安全的 SimpleDateFormat的format方法最终调用代码：

```
private StringBuffer format(Date date, StringBuffer toAppendTo,
                          FieldDelegate delegate) {
    // Convert input date to time field list
    calendar.setTime(date);

    boolean useDateFormatSymbols = useDateFormatSymbols();

    for (int i = 0; i < compiledPattern.length; ) {
        int tag = compiledPattern[i] >>> 8;
        int count = compiledPattern[i++] & 0xff;
        if (count == 255) {
            count = compiledPattern[i++] << 16;
            count |= compiledPattern[i++];
        }

        switch (tag) {
        case TAG_QUOTE_ASCII_CHAR:
            toAppendTo.append((char)count);
            break;

        case TAG_QUOTE_CHARS:
            toAppendTo.append(compiledPattern, i, count);
            i += count;
            break;

        default:
            subFormat(tag, count, delegate, toAppendTo, useDateFormatSymbols);
            break;
        }
    }
    return toAppendTo;
}
```

`calendar`是共享变量，并且这个共享变量没有做线程安全控制。

当多个线程同时使用相同的`SimpleDateFormat`对象【如用`static`修饰的SimpleDateFormat】调用format方法时。

多个线程会同时调用`calendar.setTime`方法，可能一个线程刚设置好time值，另外的一个线程马上把设置的time值给修改了，导致返回的格式化时间可能是错误的。

在`多线程`情况下使用SimpleDateFormat需格外注意 SimpleDateFormat除了format是线程不安全以外，`parse`方法也是线程不安全的。

parse方法实际调用`alb.establish(calendar).getTime()`方法来解析，alb.establish(calendar)方法里主要完成了

- 重置日期对象cal的属性值
- 使用calb中中属性设置cal
- 返回设置好的cal对象

但是这三步不是原子操作

多线程并发如何保证线程安全，避免线程之间共享一个SimpleDateFormat对象：

- 每个线程使用时都创建一次SimpleDateFormat对象 => `创建和销毁对象的开销大`
- 对使用format和parse方法的地方进行加锁 => `线程阻塞性能差`
- 使用ThreadLocal保证每个线程最多只创建一次SimpleDateFormat对象 => `较好的方法`

Date对时间处理比较麻烦，比如想获取某年、某月、某星期，以及n天以后的时间。

如果用Date来处理的话真是太难了，你可能会说Date类不是有getYear、getMonth这些方法吗，获取年月日很Easy，但都被弃用了啊

**Come On 一起使用java8全新的日期和时间API**

## LocalDate

只会获取年月日

### 创建LocalDate

```
//获取当前年月日
LocalDate localDate = LocalDate.now();
//构造指定的年月日
LocalDate localDate1 = LocalDate.of(2019, 9, 10);
```

### 获取年、月、日、星期几

```
int year = localDate.getYear();
int year1 = localDate.get(ChronoField.YEAR);
Month month = localDate.getMonth();
int month1 = localDate.get(ChronoField.MONTH_OF_YEAR);
int day = localDate.getDayOfMonth();
int day1 = localDate.get(ChronoField.DAY_OF_MONTH);
DayOfWeek dayOfWeek = localDate.getDayOfWeek();
int dayOfWeek1 = localDate.get(ChronoField.DAY_OF_WEEK);
```

## LocalTime

只会获取几点几分几秒

### 创建LocalTime

```
LocalTime localTime = LocalTime.of(13, 51, 10);
LocalTime localTime1 = LocalTime.now();
```

### 获取时分秒

```
//获取小时
int hour = localTime.getHour();
int hour1 = localTime.get(ChronoField.HOUR_OF_DAY);
//获取分
int minute = localTime.getMinute();
int minute1 = localTime.get(ChronoField.MINUTE_OF_HOUR);
//获取秒
int second = localTime.getSecond();
int second1 = localTime.get(ChronoField.SECOND_OF_MINUTE);
```

## LocalDateTime

获取年月日时分秒，等于LocalDate+LocalTime

### 创建LocalDateTime

```
LocalDateTime localDateTime = LocalDateTime.now();
LocalDateTime localDateTime1 = LocalDateTime.of(2019, Month.SEPTEMBER, 10, 14, 46, 56);
LocalDateTime localDateTime2 = LocalDateTime.of(localDate, localTime);
LocalDateTime localDateTime3 = localDate.atTime(localTime);
LocalDateTime localDateTime4 = localTime.atDate(localDate);
```

### 获取LocalDate

```
LocalDate localDate2 = localDateTime.toLocalDate();
```

### 获取LocalTime

```
LocalTime localTime2 = localDateTime.toLocalTime();
```

## Instant

获取秒数

### 创建Instant对象

```
Instant instant = Instant.now();
```

### 获取秒数

```
long currentSecond = instant.getEpochSecond();
```

### 获取毫秒数

```
long currentMilli = instant.toEpochMilli();
```

个人觉得如果只是为了获取秒数或者毫秒数，使用System.currentTimeMillis()来得更为方便

### 修改LocalDate、LocalTime、LocalDateTime、Instant

LocalDate、LocalTime、LocalDateTime、Instant为不可变对象，修改这些对象对象会返回一个副本

增加、减少年数、月数、天数等 以LocalDateTime为例：

```
LocalDateTime localDateTime = LocalDateTime.of(2019, Month.SEPTEMBER, 10, 14, 46, 56);
//增加一年
localDateTime = localDateTime.plusYears(1);
localDateTime = localDateTime.plus(1, ChronoUnit.YEARS);
//减少一个月
localDateTime = localDateTime.minusMonths(1);
localDateTime = localDateTime.minus(1, ChronoUnit.MONTHS);  
```

通过with修改某些值

```
//修改年为2019
localDateTime = localDateTime.withYear(2020);
//修改为2022
localDateTime = localDateTime.with(ChronoField.YEAR, 2022);
```

还可以修改月、日

### 时间计算

比如有些时候想知道这个月的最后一天是几号、下个周末是几号，通过提供的时间和日期API可以很快得到答案

比如通过firstDayOfYear()返回了当前日期的第一天日期，还有很多方法这里不在举例说明

```
LocalDate localDate = LocalDate.now();
LocalDate localDate1 = localDate.with(firstDayOfYear());
```

### 格式化时间

```
LocalDate localDate = LocalDate.of(2019, 9, 10);
String s1 = localDate.format(DateTimeFormatter.BASIC_ISO_DATE);
String s2 = localDate.format(DateTimeFormatter.ISO_LOCAL_DATE);
//自定义格式化
DateTimeFormatter dateTimeFormatter =   DateTimeFormatter.ofPattern("dd/MM/yyyy");
String s3 = localDate.format(dateTimeFormatter);
```

DateTimeFormatter默认提供了多种格式化方式，如果默认提供的不能满足要求，可以通过DateTimeFormatter的ofPattern方法创建自定义格式化方式

### 解析时间

```
LocalDate localDate1 = LocalDate.parse("20190910", DateTimeFormatter.BASIC_ISO_DATE);
LocalDate localDate2 = LocalDate.parse("2019-09-10", DateTimeFormatter.ISO_LOCAL_DATE);
```

和SimpleDateFormat相比，DateTimeFormatter是线程安全的

## 小结

`LocalDateTime`：Date有的我都有，Date没有的我也有，日期选择请Pick Me

SpringBoot中应用LocalDateTime

将LocalDateTime字段以时间戳的方式返回给前端 添加日期转化类

```
public class LocalDateTimeConverter extends JsonSerializer<LocalDateTime> {

    @Override
    public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
    gen.writeNumber(value.toInstant(ZoneOffset.of("+8")).toEpochMilli());
    }
}
```

并在LocalDateTime字段上添加`@JsonSerialize(using = LocalDateTimeConverter.class`)注解，如下：

```
@JsonSerialize(using = LocalDateTimeConverter.class)
protected LocalDateTime gmtModified;
```

将LocalDateTime字段以指定格式化日期的方式返回给前端。

在LocalDateTime字段上添加`@JsonFormat(shape=JsonFormat.Shape.STRING, pattern="yyyy-MM-dd HH:mm:ss")`注解即可，如下：

```
@JsonFormat(shape=JsonFormat.Shape.STRING, pattern="yyyy-MM-dd HH:mm:ss")
protected LocalDateTime gmtModified;
```

对前端传入的日期进行格式化 在`LocalDateTime`字段上添加`@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")`注解即可，如下：

```
@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
protected LocalDateTime gmtModified;
```

adasd