---
layout: post
title:  "线程不安全的 SimpleDateFormat"
date:   2017-12-06 17:12:12 +0800
categories: java
---
从 [JavaDoc][] 可以查到，`SimpleDateFormat` 是一个与本地操作系统环境有关的用于格式化和解析日期的工具类，能够实现从日期到文本的格式化，文本到日期的解析，以及规范化等功能。

`java.text.SimpleDateFormat` 继承自 `java.text.DateFormat`，而 `java.text.DateFormat` 继承自 `java.text.Format`。值得注意的是，这三个类都不是线程安全的。关于同步，[JavaDoc][] 写道：

> #### Synchronization
> Date formats are not synchronized. It is recommended to create separate format instances for each thread. If multiple threads access a format concurrently, it must be synchronized externally.

也就是说，`SimpleDateFormat` 是线程不安全的，使用时应当给每一个线程创建对应的单独的实例，或者同步访问同一个实例。具体来说，一种单线程的典型的用法如下。

    private static final DateFormat df = new SimpleDateFormat("yyyyMMddHHmmss");

    private void parse() throws ParseException {
        Date date = df.parse("20171206105413");
        System.out.println(date);
    }

以上程序可以正常运行，但是，它不是线程安全的，通过多线程就能够测试出来。

    private void run() throws ParseException, ExecutionException, InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(2);

        Callable<Date> task = new Callable<Date>() {
            public Date call() throws Exception {
                return parse();
            }
        };

        List<Future<Date>> results = new ArrayList<Future<Date>>();

        for (int i = 0; i < 5; i++) {
            results.add(executorService.submit(task));
        }

        executorService.shutdown();

        boolean flag = executorService.awaitTermination(1, TimeUnit.MINUTES);
        if (flag) {
            for (Future<Date> result : results) {
                System.out.println(result.get());
            }
        }
    }

以上程序通过 `run()` 并行调用 `parse()`，其运行结果是不可预测的，几种可能结果如下。

    Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.NumberFormatException: multiple points

    Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.NumberFormatException: For input string: "201E20140"

    Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.NumberFormatException: empty String

    Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.ArrayIndexOutOfBoundsException: -1

    Sat Dec 06 10:54:13 CST 2200
    Sat Dec 06 10:54:13 CST 2200
    Wed Dec 06 10:54:13 CST 2017
    Tue Jun 06 10:54:13 CST 2017
    Wed Dec 06 10:54:13 CST 2017

#### 为什么 `SimpleDateFormat` 是线程不安全的？

因为 `SimpleDateFormat` 使用方法局部变量保存中间计算结果，所以当使用多线程调用格式化或解析方法时就会出现错误的结果。比如 `parse()` 方法，使用了 `CalendarBuilder` 局部变量来操作数据，先调用 `clear()`，再调用 `addYear()`，当一个线程在调用了第一个方法之后，另外一个线程调用了 `parse()` 时就会产生混乱。

#### 怎样正确地使用 `SimpleDateFormat`？

有几种方法可以线程安全地使用 DateFormat，详细代码示例参见 [simpledateformat-demo][]。

1. 使用 `synchronized` 同步

        private Date parse() throws ParseException {
            synchronized (df) {
                Date date = df.parse("20171206105413");
                return date;
            }
        }

2. 使用 `ThreadLocal` 同步

        private final ThreadLocal<DateFormat> df = new ThreadLocal<DateFormat>() {
            protected DateFormat initialValue() {
                return new SimpleDateFormat("yyyyMMddHHmmss");
            }
        };

3. 使用 [joda-time][] 工具类

        import org.joda.time.format.DateTimeFormat;
        import org.joda.time.format.DateTimeFormatter;

        private static final DateTimeFormatter df = DateTimeFormat.forPattern("yyyyMMddHHmmss");

#### 参考文献

1. [Why is Java's SimpleDateFormat not thread-safe?][why]
2. [“Java DateFormat is not threadsafe” what does this leads to?][what]
3. [DateFormat with Multiple Threads][how]

[JavaDoc]: https://docs.oracle.com/javase/6/docs/api/java/text/SimpleDateFormat.html
[joda-time]: http://www.joda.org/joda-time/
[why]: https://stackoverflow.com/questions/6840803/why-is-javas-simpledateformat-not-thread-safe
[what]: https://stackoverflow.com/questions/4021151/java-dateformat-is-not-threadsafe-what-does-this-leads-to
[how]: http://fahdshariff.blogspot.com/2010/08/dateformat-with-multiple-threads.html
[simpledateformat-demo]: https://github.com/fishkq/simpledateformat-demo
