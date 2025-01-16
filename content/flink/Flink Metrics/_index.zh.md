+++
categories = ["flink"]
tags = ["flink", "metrics"]
description = "Push custom metrics to prometheus."
title = "Flink Metrics"
weight = 1
+++

以下内容只说明了flink指标系统的常用功能，关于flink指标的更详细的信息:

1. [flink1.19指标系统官方文档](https://nightlies.apache.org/flink/flink-docs-release-1.9/monitoring/metrics.html)
2. [flink-k8s-operator 指标系统官方文档](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-release-1.10/docs/operations/metrics-logging/)

## 概述
flink的指标系统只能在RichFunction种访问。flink共支持4种指标： Counters, Gauges, Histograms 以及 Meters。flink的指标类型和prometheus的指标类型有差异，访问[Prometheus指标类型](https://www.yuque.com/useless-fajkd/gny0lq/ntbfe9z1r0zgpi0t?singleDoc#%20《Prometheus指标类型》)了解Prometheus指标的分类。flink的指标类型概述见下面的表格：

| Flink指标 | 对应Prometheus指标 | 用途 | <font style="color:#DF2A3F;">注意</font> |
| --- | --- | --- | --- |
| Counter | Gauge | 计数，比如总请求数 | flink Counter可以增也可以减，Prometheus只可以增 |
| Gauge | Gauge | 记录瞬时值，如cpu使用率 | |
| Histogram | Summary | 提供统计信息，如时延最大 最小 平均 95 99 | flink本身不提供实现，需要引入额外依赖，下方具体指标的使用有使用说明 |
| Meter | Gauge | 提供吞吐量统计 | flink本身不提供实现，需要引入额外依赖，下方具体指标的使用有使用说明 |


## Counter
counter指标用于计数，比如总的请求数量。可以通过调用inc()/inc(long n) or dec()/dec(long n)来对一个counter指标进行1或其它数量的增减。

这里模拟一个counter指标，在紧接source的mapFunction里对source发送的记录进行计数：

```java
package com.zhejianglab.astronomy;

import org.apache.flink.api.common.functions.OpenContext;
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.metrics.Counter;

public class CounterDemoMapFunction extends RichMapFunction<Long, Long> {
    private transient Counter counter;

    @Override
    public void open(OpenContext openContext) throws Exception {
        super.open(openContext);
        this.counter = getRuntimeContext()
                .getMetricGroup()
                .counter("Record_Counter");
    }

    @Override
    public Long map(Long value) throws Exception {
        this.counter.inc();
        return value;
    }
}

```



定义好后，在flink web 可以查看这个指标

![](https://cdn.nlark.com/yuque/0/2025/png/38446785/1736756699390-293b993f-87c9-4804-ba68-bd9ab400bf64.png)

部署到k8s，可以到Prometheus查看这个指标

![](https://cdn.nlark.com/yuque/0/2025/png/38446785/1736818643901-4014f135-f1f1-410c-84f7-c8c4da174e06.png)

## Guage
guage指标用于暂时某些瞬时的值，典型的比如cpu、内存使用率。当不关心统计信息，只关心当前的值是可以使用这个指标。

这里模拟一个guage指标，在sink里对记录从source到sink的延迟进行记录(记录本身就是从source里生成时的时时间戳)，这里这个指标的含义是“最近一个到达sink的记录的延迟”，是一个瞬时值：

```java
package com.zhejianglab.astronomy;

import org.apache.flink.api.common.functions.OpenContext;
import org.apache.flink.metrics.Gauge;
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;

import java.util.concurrent.TimeUnit;

public class MockSink extends RichSinkFunction<Long> {
    private transient long delay = 0;

    @Override
    public void open(OpenContext openContext) throws Exception {
        super.open(openContext);
        getRuntimeContext()
                .getMetricGroup()
                .gauge("Delay_Gauge", new Gauge<Long>() {
                    @Override
                    public Long getValue() {
                        return delay;
                    }
                });
    }

    @Override
    public void invoke(Long value, Context context) throws Exception {
        this.delay = System.currentTimeMillis() - value;
        System.out.println(value);
    }

    @Override
    public void finish() throws Exception {
        super.finish();
    }
}

```



定义好后，在flink web 可以查看这个指标:

![](https://cdn.nlark.com/yuque/0/2025/png/38446785/1736756727477-5d6936fc-3c00-4194-8728-c00c55586515.png)



## <font style="color:#DF2A3F;">注意</font>
下面要介绍的Histogram和Meter指标，flink本身只定义了接口，没有提供具体的实现。要使用这两个指标，需要引入依赖

```kotlin
mplementation("org.apache.flink:flink-metrics-dropwizard:$flinkVersion")
```

之后可wrap为flink接口的实现，具体使用方法参考下面具体代码。

## Histogram
对应上面的Guage，如果我们想在记录延迟时同时记录延迟的统计信息，比如最大最小平均99线等，这时候可以使用histogram指标。

这里模拟一个histogram指标，在sink里对记录从source到sink的延迟进行记录(记录本身就是从source里生成时的时时间戳)，这里这个指标的含义是“10秒窗口内记录从source到sink的记录的延迟的统计信息”，实际是多个指标的集合：

```java
package com.zhejianglab.astronomy;

import com.codahale.metrics.SlidingTimeWindowReservoir;
import org.apache.flink.api.common.functions.OpenContext;
import org.apache.flink.dropwizard.metrics.DropwizardHistogramWrapper;
import org.apache.flink.metrics.Histogram;
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;

import java.util.concurrent.TimeUnit;

public class MockSink extends RichSinkFunction<Long> {
    private transient long delay = 0;

    private transient Histogram histogram;
    @Override
    public void open(OpenContext openContext) throws Exception {
        super.open(openContext);
       
        com.codahale.metrics.Histogram dropwizardHistogram =
                new com.codahale.metrics.Histogram(new SlidingTimeWindowReservoir(10, TimeUnit.SECONDS));

        this.histogram = getRuntimeContext()
                .getMetricGroup()
                .histogram("Delay_Histogram", new DropwizardHistogramWrapper(dropwizardHistogram));
    }

    @Override
    public void invoke(Long value, Context context) throws Exception {
        this.delay = System.currentTimeMillis() - value;
        this.histogram.update(this.delay);
        System.out.println(value);
    }

    @Override
    public void finish() throws Exception {
        super.finish();
    }
}

```



定义好后，在flink web 可以查看这个指标实际生成了多个指标：

![](https://cdn.nlark.com/yuque/0/2025/png/38446785/1736756839940-d23efc73-d28c-40be-95d9-9f7f964c617e.png)

选取99线进行查看：

![](https://cdn.nlark.com/yuque/0/2025/png/38446785/1736756947688-298bb696-1adb-457a-9d44-7bd310214038.png)



## Meter
meter是一个用来统计吞吐量的指标，比如在紧接source的mapFunction里对source发送的记录进行吞吐量统计：

```java
package com.zhejianglab.astronomy;

import org.apache.flink.api.common.functions.OpenContext;
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.dropwizard.metrics.DropwizardMeterWrapper;
import org.apache.flink.metrics.Meter;

public class CounterDemoMapFunction extends RichMapFunction<Long, Long> {
    private transient Meter meter;

    @Override
    public void open(OpenContext openContext) throws Exception {
        super.open(openContext);

        com.codahale.metrics.Meter dropwizardMeter = new com.codahale.metrics.Meter();

        this.meter = getRuntimeContext()
                .getMetricGroup()
                .meter("Record_Meter", new DropwizardMeterWrapper(dropwizardMeter));
    }

    @Override
    public Long map(Long value) throws Exception {
        this.meter.markEvent();
        return value;
    }
}

```



定义好后，在flink web 可以查看这个指标，source每秒发送12-13个记录过来：

![](https://cdn.nlark.com/yuque/0/2025/png/38446785/1736756792723-3b999a99-8d28-48b6-8b76-5fe69fb6a26b.png)



## Flink完整例子
![](https://cdn.nlark.com/yuque/0/2025/png/38446785/1736759077335-f68ce21d-de47-4e68-a952-229cb9a4f16d.png)

```kotlin
plugins {
    java
    id("io.github.goooler.shadow") version "8.1.8"
}

val lombokDependency = "org.projectlombok:lombok:1.18.22"
var group = "com.zhejianglab.astronomy"
var version = "1.0-SNAPSHOT"
var flinkVersion = "1.19.1"
var slf4jVersion = "2.0.9"
var logbackVersion = "1.4.14"
val jacksonVersion = "2.13.4"

repositories {
    mavenCentral()
}

dependencies {
    annotationProcessor(lombokDependency)

    implementation("org.apache.flink:flink-metrics-dropwizard:$flinkVersion")

    implementation(lombokDependency)
    implementation("org.apache.flink:flink-s3-fs-hadoop:$flinkVersion")
    implementation("org.slf4j:slf4j-simple:$slf4jVersion")
    implementation("org.apache.flink:flink-streaming-java:$flinkVersion")
    implementation("org.apache.flink:flink-clients:$flinkVersion")
    implementation("org.apache.flink:flink-runtime-web:$flinkVersion")
    testImplementation(platform("org.junit:junit-bom:5.9.1"))
    testImplementation("org.junit.jupiter:junit-jupiter")
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(17))
    }
}

tasks.test {
    useJUnitPlatform()
}
```



```java
package com.zhejianglab.astronomy;

import org.apache.flink.configuration.Configuration;
import org.apache.flink.configuration.RestOptions;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class Main {
    public static void main(String[] args) throws Exception {
        Configuration configuration = new Configuration();
        configuration.set(RestOptions.PORT, 8081);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(configuration);
        env.setParallelism(1);

        env.addSource(new MockSource()).map(new CounterDemoMapFunction()).addSink(new MockSink());

        env.execute("flink-metrics");
    }
}
```



```java
package com.zhejianglab.astronomy;

import org.apache.flink.api.common.functions.OpenContext;
import org.apache.flink.streaming.api.functions.source.RichSourceFunction;

import java.util.Random;

public class MockSource extends RichSourceFunction<Long> {
    private long startTime;

    private transient Random random;
    @Override
    public void open(OpenContext openContext) throws Exception {
        super.open(openContext);
        random = new Random();
        this.startTime = System.currentTimeMillis();
    }

    @Override
    public void run(SourceContext<Long> ctx) throws Exception {
        while(System.currentTimeMillis() - startTime < 1000 * 60 * 20) {
            long currentTs = System.currentTimeMillis();
            Thread.sleep( random.nextInt(150) + 1);
            ctx.collect(currentTs);
        }
    }

    @Override
    public void cancel() {

    }
}

```



```java
package com.zhejianglab.astronomy;

import org.apache.flink.api.common.functions.OpenContext;
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.dropwizard.metrics.DropwizardMeterWrapper;
import org.apache.flink.metrics.Counter;
import org.apache.flink.metrics.Meter;

public class CounterDemoMapFunction extends RichMapFunction<Long, Long> {
    private transient Counter counter;
    private transient Meter meter;

    @Override
    public void open(OpenContext openContext) throws Exception {
        super.open(openContext);
        this.counter = getRuntimeContext()
                .getMetricGroup()
                .counter("Record_Counter");

        com.codahale.metrics.Meter dropwizardMeter = new com.codahale.metrics.Meter();

        this.meter = getRuntimeContext()
                .getMetricGroup()
                .meter("Record_Meter", new DropwizardMeterWrapper(dropwizardMeter));
    }

    @Override
    public Long map(Long value) throws Exception {
        this.counter.inc();
        this.meter.markEvent();
        return value;
    }
}
```



```java
package com.zhejianglab.astronomy;

import com.codahale.metrics.SlidingTimeWindowReservoir;
import org.apache.flink.api.common.functions.OpenContext;
import org.apache.flink.dropwizard.metrics.DropwizardHistogramWrapper;
import org.apache.flink.metrics.Gauge;
import org.apache.flink.metrics.Histogram;
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;

import java.util.concurrent.TimeUnit;

public class MockSink extends RichSinkFunction<Long> {
    private transient long delay = 0;

    private transient Histogram histogram;
    @Override
    public void open(OpenContext openContext) throws Exception {
        super.open(openContext);
        getRuntimeContext()
                .getMetricGroup()
                .gauge("Delay_Gauge", new Gauge<Long>() {
                    @Override
                    public Long getValue() {
                        return delay;
                    }
                });

        com.codahale.metrics.Histogram dropwizardHistogram =
                new com.codahale.metrics.Histogram(new SlidingTimeWindowReservoir(10, TimeUnit.SECONDS));

        this.histogram = getRuntimeContext()
                .getMetricGroup()
                .histogram("Delay_Histogram", new DropwizardHistogramWrapper(dropwizardHistogram));
    }

    @Override
    public void invoke(Long value, Context context) throws Exception {
        this.delay = System.currentTimeMillis() - value;
        this.histogram.update(this.delay);
        System.out.println(value);
    }

    @Override
    public void finish() throws Exception {
        super.finish();
    }
}

```
