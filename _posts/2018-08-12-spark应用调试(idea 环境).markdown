---
layout: post
title:  "spark应用调试(idea 环境)"
date:   2018-08-12 11:42:19 +0800
categories: spark
---
### spark应用调试(idea 环境)

spark shell可以比较方便的分步执行调试spark应用程序，但有时候不是很方便。下面介绍一种直接在idea中调试spark程序的方法。

#### maven 新建工程

​      这个不多赘述，注意一点是pom文件中把依赖的spark的scope的provided去掉，因为我们要在idea中直接运行，不会用spark-submit。

```
<dependencies>
    <dependency>
        <groupId>org.scala-lang</groupId>
        <artifactId>scala-library</artifactId>
        <version>${scala.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.11</artifactId>
        <version>${spark.version}</version>
        <!--<scope>provided</scope>-->
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql_2.11</artifactId>
        <version>${spark.version}</version>
        <!--<scope>provided</scope>-->
    </dependency>
</dependencies>
```

```
<properties>
    <scala.version>2.11.2</scala.version>
    <spark.version>2.3.1</spark.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties>
```

#### 编写应用

```
package neal.spark

import org.apache.spark.sql.SparkSession

import scala.math.random

/** Computes an approximation to pi */
object SparkPi {
  def main(args: Array[String]) {
    val spark = SparkSession
      .builder
      .appName("Spark Pi")
      .master("local")
      .getOrCreate()
    val slices = if (args.length > 0) args(0).toInt else 2
    val n = math.min(100000L * slices, Int.MaxValue).toInt // avoid overflow
    val count = spark.sparkContext.parallelize(1 until n, slices).map { i =>
      val x = random * 2 - 1
      val y = random * 2 - 1
      if (x*x + y*y <= 1) 1 else 0
    }.reduce(_ + _)
    println(s"Pi is roughly ${4.0 * count / (n - 1)}")
    spark.stop()
  }
}
```

以spark pi为例，这里注意要加上**.master("local")**，在local模式运行

#### 运行调试

配置一个run configure 就可以运行调试了，简单快捷。

idea会自动帮你下载source，自己标记断点即可。

如下图所示，在spark.sparkContext.parallelize处添加断点：

![屏幕快照 2018-07-15 下午5.41.41](https://ws3.sinaimg.cn/large/006tKfTcly1ftaosktv83j31iw0w6k2g.jpg)

#### 小技巧

1. 将log4j.propertities设置为trace level，spark会打印更多的调试信息

```
#配置根Logger 后面是若干个Appender
#log4j.rootLogger=DEBUG,A1,R,E

log4j.rootLogger=TRACE,A1

# ConsoleAppender 输出
log4j.appender.A1=org.apache.log4j.ConsoleAppender
log4j.appender.A1.layout=org.apache.log4j.PatternLayout
log4j.appender.A1.layout.ConversionPattern=%-d{yyyy-MM-dd HH:mm:ss,SSS} [%c]-[%p] %m%n
```

2. 调试 spark sql程序，打印 tree 信息
   ```
   println(spark.sql("select * from a").queryExecution.analyzed.treeString)
   ```

`+- LocalLimit 21`
   +- AnalysisBarrier
         +- Project [cast(age#6L as string) AS age#12, cast(name#7 as string) AS name#13]
            `+- Relation[age#6L,name#7] json`