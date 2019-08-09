spring boot内置容器性能比较(Jetty、Tomcat、Undertow)
====
一、准备工作
-------
### 1.1 服务器环境

| 名称               |配置                                      |
| --                | --                                      |
| 服务器操作系统       |Ubuntu 18.04.2 LTS                       |
|内存                |15.4 GiB                                 |
|处理器              | Intel® Core™ i7-7560U CPU @ 2.40GHz × 4 |
|磁盘                |SSD 177.2 GB                             |
|jdk                |openjdk-1.8.0_222                        |
|spring boot        |2.1.7.RELEASE                            |
|tomcat             |9.0.22                                   |
|jetty              |9.4.19.v20190610                         |
|undertow           |2.0.23.Final                             |
### 1.2 创建服务

新建spring boot项目

1. jetty
2. tomcat
3. undertow

### 1.3 安装相关性能测试工具

安装 visualvm

``` shell
sudo apt-get install visualvm
```

安装visualgc(visualvm的插件)安装visualgc
可在VisualVM的**Tools**->**Plugins**->**AvailablePlugins**里边找到Visual GC插件进行安装
重启后可见到相关tab

安装 ab(ApacheBench)测试工具

``` shell
sudo apt-get install apache2-utils
```

安装 siege 高性能压测工具

``` shell
sudo apt-get install siege 
```
二、服务设置
------

这边每个服务具体初始化配置如下

### 2.1 tomcat

``` pom
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### 2.2 jetty

``` pom
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jetty</artifactId>
    </dependency>
</dependencies>
```

### 2.3 undertow

``` pom
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-undertow</artifactId>
    </dependency>
</dependencies>
```
### 2.4 每个服务rest服务配置如下

``` java
package com.songyaxu.container.jetty.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.request.async.WebAsyncTask;

/**
 * @Author： yaxuSong
 * @Description：
 * @Date： 19-8-8 上午11:41
 * @MOdified by:
 **/
@RestController
@RequestMapping("test")
public class TestController {


    @Value("${server.name}")
    private String SERVER_NAME;

    /**
     * 未使用HTTP异步的接口
     */
    @GetMapping("/block")
    public String block() {
        return SERVER_NAME + " block!!!";
    }

    /**
     * 使用HTTP异步的接口
     */
    @GetMapping("/async")
    public WebAsyncTask<String> async() {
        return new WebAsyncTask(() -> SERVER_NAME + " async!!!");
    }
}


```
三、进行测试
--------

### 3.1 第一轮简单的测试

###### 使用seige进行的http同步测试

访问情况统计

|服务器   |条件        |命中   |成功率|吞吐量            |平均耗时|
|---     |---        |---   |---  |---              |---   |
|tomcat  |-c 50  -t 2|197626|100% |1656.41 trans/sec|0.03  |
|tomcat  |-c 100 -t 2|201692|100% |1690.35 trans/sec|0.06  |
|tomcat  |-c 200 -t 2|240596|100% |2010.66 trans/sec|0.10  |
|jetty   |-c 50  -t 2|282701|100% |2364.51 trans/sec|0.03  |
|jetty   |-c 100 -t 2|319180|100% |2674.10 trans/sec|0.04  |
|jetty   |-c 200 -t 2|224428|100% |1882.79 trans/sec|0.11  |
|undertow|-c 50  -t 2|260263|100% |2176.11 trans/sec|0.02  |
|undertow|-c 100 -t 2|349105|100% |2925.54 trans/sec|0.03  |
|undertow|-c 200 -t 2|358812|100% |3006.38 trans/sec|0.07  |

开启线程统计

|服务器   |条件		   |存活线程数|守护线程数|累计开启线程数|
|---     |---        |---     |---     | ---         |
|tomcat  |初始        |   29   |    25  |42           |
|tomcat  |-c 50  -t 2|   93   |    89  |111          |
|tomcat  |-c 100 -t 2|   149  |    145 |365          |
|tomcat  |-c 200 -t 2|   196  |    192 |532          |
|jetty   |初始        |   22   |     12 |35           |
|jetty   |-c 50  -t 2|  132   |    11  |147          |
|jetty   |-c 100 -t 2|  173   |    11  |195          |
|jetty   |-c 200 -t 2|  215   |    12  |239          |
|undertow|初始        |  18    |    12  |32           |
|undertow|-c 50  -t 2|  50    |    12  |65           |
|undertow|-c 100 -t 2|  50    |    12  |65           |
|undertow|-c 200 -t 2|  50    |    12  |67           |

cpu和内存统计

|服务器   |条件        |CPU占用率|内存占用率|最高cpu占用率|
|---     |---        |---     |---     | ---       |
|tomcat  |初始        |  1.0%  |  1.8%  | 2.3%      |
|tomcat  |-c 50  -t 2|  56.1% |  3.0%  | 250.3%    |
|tomcat  |-c 100 -t 2|  53.1% |  3.7%  | 211.3%    |
|tomcat  |-c 200 -t 2|  53.6% |  2.8%  | 222.7%    |
|jetty   |初始        |  0.3%  |  1.5%  | 1.5%      |
|jetty   |-c 50  -t 2|  67.2% |  5.7%  |  204.0%   |
|jetty   |-c 100 -t 2|  55.9% |  5.2%  |211.7%     |
|jetty   |-c 200 -t 2|  54.7% |  4.2%  |  210.3%   |
|undertow|初始        |  0.3%  |  1.5%  | 1.5%      |
|undertow|-c 50  -t 2|  64.3% |  4.7%  |   190.1%  |
|undertow|-c 100 -t 2|  50.9% |  4.4%  |  197.3%   |
|undertow|-c 200 -t 2|  51.5% |  3.9%  |  198.3%   |

###### 使用seige进行的http异步测试

访问情况统计

|服务器   |条件        |命中   |成功率|吞吐量            |平均耗时|
|---     |---        |---   |---  |---              |---   |
|tomcat  |-c 50  -t 2|128260|100% |1076.28 trans/sec|0.05  |
|tomcat  |-c 100 -t 2|145973|100% |1218.47 trans/sec|0.08  |
|tomcat  |-c 200 -t 2|161967|100% |1356.62 trans/sec|0.15  |
|jetty   |-c 50  -t 2|173843|100% |1459.88 trans/sec|0.03  |
|jetty   |-c 100 -t 2|189112|100% |1586.51 trans/sec|0.06  |
|jetty   |-c 200 -t 2|202334|100% |1699.14 trans/sec|0.12  |
|undertow|-c 50  -t 2|201798|100% |1684.60 trans/sec|0.03  |
|undertow|-c 100 -t 2|217834|100% |1819.53 trans/sec|0.05  |
|undertow|-c 200 -t 2|248644|100% |2081.40 trans/sec|0.10  |

开启线程统计

|服务器   |条件         |存活线程数|守护线程数|累计开启线程数|
|---     |---        |---     |---     | ---         |
|tomcat  |初始        |   33   |    29  |43           |
|tomcat  |-c 50  -t 2|   102  |    90  |116          |
|tomcat  |-c 100 -t 2|   160  |    148 |251          |
|tomcat  |-c 200 -t 2|   226  |    214 |450          |
|jetty   |初始        |   27   |     17 |37           |
|jetty   |-c 50  -t 2|  122   |    11  |148          |
|jetty   |-c 100 -t 2|  182   |    12  |208          |
|jetty   |-c 200 -t 2|  223   |    12  |260          |
|undertow|初始        |  20    |    14  |31           |
|undertow|-c 50  -t 2|  49    |    12  |71           |
|undertow|-c 100 -t 2|  57    |    11  |80           |
|undertow|-c 200 -t 2|  49    |    12  |88           |

cpu和内存统计

|服务器   |条件        |CPU占用率|内存占用率|最高cpu占用率|
|---     |---        |---     |---     | ---       |
|tomcat  |初始        |  0.3%  |  1.5%  | 1.5%      |
|tomcat  |-c 50  -t 2|  76.1% |  6.9%  | 231.0%    |
|tomcat  |-c 100 -t 2|  60.3% |  8.3%  | 231.3%    |
|tomcat  |-c 200 -t 2|  58.1% |  9.0%  | 230.2%    |
|jetty   |初始        |  1.0%  |  1.2%  | 1.3%      |
|jetty   |-c 50  -t 2|  75.1% |  3.7%  |  227.3%   |
|jetty   |-c 100 -t 2|  59.9% |  3.1%  |  228.7%   |
|jetty   |-c 200 -t 2|  60.3% |  3.1%  |  226.1%   |
|undertow|初始        |  0.7%  |  1.5%  | 1.5%      |
|undertow|-c 50  -t 2|  66.4% |  3.5%  |  213.7%   |
|undertow|-c 100 -t 2|  56.9% |  2.9%  |  219.3%   |
|undertow|-c 200 -t 2|  55.0% |  2.5%  |  215.7%   |

一些对比的图片

-![tomcat-async-moitor](https://weaf.oss-cn-beijing.aliyuncs.com/tomcat-async-moitor.png)
-![tomcat-async-gc](https://weaf.oss-cn-beijing.aliyuncs.com/tomcat-async-gc.png)
-![jetty-async-monitor](https://weaf.oss-cn-beijing.aliyuncs.com/jetty-async-monitor.png)
-![jetty-async-gc](https://weaf.oss-cn-beijing.aliyuncs.com/jetty-async-gc.png)
-![undertow-async-monitor](https://weaf.oss-cn-beijing.aliyuncs.com/undertow-async-monitor.png)
-![undertow-async-gc](https://weaf.oss-cn-beijing.aliyuncs.com/undertow-async-gc.png)


### 3.2 第二轮简单的测试

###### 使用ab进行的http同步测试

访问情况统计

|服务器   |条件            |成功率|吞吐量           |平均耗时|总消耗时间|
|---     |---            |---  |---             |---   |---     |
|tomcat  |-n 20000 -c 50 |100% |10739.60 [#/sec]|0.093 |1.862   |
|tomcat  |-n 20000 -c 100|100% |11437.54 [#/sec]|0.087 |1.749   |
|tomcat  |-n 20000 -c 200|100% |12582.45 [#/sec]|0.079 |1.590   |
|jetty   |-n 20000 -c 50 |100% |8546.79  [#/sec]|0.117 |2.340   |
|jetty   |-n 20000 -c 100|100% |10445.64 [#/sec]|0.096 |1.915   |
|jetty   |-n 20000 -c 200|100% |10777.76 [#/sec]|0.093 |1.856   |
|undertow|-n 20000 -c 50 |100% |8801.86  [#/sec]|0.114 |2.272   |
|undertow|-n 20000 -c 100|100% |11803.92 [#/sec]|0.085 |1.694   |
|undertow|-n 20000 -c 200|100% |12604.54 [#/sec]|0.079 |1.587   |

开启线程统计

|服务器   |条件            |存活线程数|守护线程数|累计开启线程数|
|---     |---            |---     |---     | ---         |
|tomcat  |-n 20000 -c 50 |   85   |    81  |95           |
|tomcat  |-n 20000 -c 100|   122  |    118 |189          |
|tomcat  |-n 20000 -c 200|   139  |    135 |300          |
|jetty   |-n 20000 -c 50 |  129   |    12  |145          |
|jetty   |-n 20000 -c 100|  140   |    12  |158          |
|jetty   |-n 20000 -c 200|  214   |    11  |233          |
|undertow|-n 20000 -c 50 |  52    |    14  |63           |
|undertow|-n 20000 -c 100|  50    |    12  |63           |
|undertow|-n 20000 -c 200|  50    |    12  |63           |

cpu和内存统计

|服务器   |条件            |CPU占用率|内存占用率|
|---     |---            |---     |---     |
|tomcat  |-n 20000 -c 50 |  59.8% |  2.7%  |
|tomcat  |-n 20000 -c 100|  58.2% |  3.5%  |
|tomcat  |-n 20000 -c 200|  53.2% |  3.7%  |
|jetty   |-n 20000 -c 50 |  58.7% |  1.7%  |
|jetty   |-n 20000 -c 100|  58.6% |  2.2%  |
|jetty   |-n 20000 -c 200|  64.1% |  2.3%  |
|undertow|-n 20000 -c 50 |  59.2% |  2.3%  |
|undertow|-n 20000 -c 100|  64.3% |  2.7%  |
|undertow|-n 20000 -c 200|  57.9% |  3.6%  |

###### 使用ab进行的http异步测试

访问情况统计

|服务器   |条件            |成功率|吞吐量           |平均耗时|总消耗时间|
|---     |---            |---  |---             |---   |---     |
|tomcat  |-n 20000 -c 50 |100% |5870.13  [#/sec]|0.170 |3.407   |
|tomcat  |-n 20000 -c 100|100% |8311.58  [#/sec]|0.120 |1.749   |
|tomcat  |-n 20000 -c 200|100% |8774.37  [#/sec]|0.114 |2.279   |
|jetty   |-n 20000 -c 50 |100% |6519.73  [#/sec]|0.153 |3.068   |
|jetty   |-n 20000 -c 100|100% |8270.94  [#/sec]|0.121 |2.418   |
|jetty   |-n 20000 -c 200|100% |8530.12  [#/sec]|0.117 |2.345   |
|undertow|-n 20000 -c 50 |100% |6659.26  [#/sec]|0.150 |3.003   |
|undertow|-n 20000 -c 100|100% |9329.74  [#/sec]|0.107 |2.144   |
|undertow|-n 20000 -c 200|100% |9856.14  [#/sec]|0.101 |2.029   |

开启线程统计

|服务器   |条件            |存活线程数|守护线程数|累计开启线程数|
|---     |---            |---     |---     | ---         |
|tomcat  |-n 20000 -c 50 |   89   |    77  |99           |
|tomcat  |-n 20000 -c 100|   140  |    128 |210          |
|tomcat  |-n 20000 -c 200|   183  |    195 |376          |
|jetty   |-n 20000 -c 50 |  131   |    16  |142          |
|jetty   |-n 20000 -c 100|  153   |    12  |177          |
|jetty   |-n 20000 -c 200|  184   |    12  |226          |
|undertow|-n 20000 -c 50 |  61    |    17  |73           |
|undertow|-n 20000 -c 100|  58    |    12  |81           |
|undertow|-n 20000 -c 200|  58    |    12  |89           |

cpu和内存统计

|服务器   |条件            |CPU占用率|内存占用率|
|---     |---            |---     |---     |
|tomcat  |-n 20000 -c 50 |  71.8% |  2.9%  |
|tomcat  |-n 20000 -c 100|  65.2% |  4.3%  |
|tomcat  |-n 20000 -c 200|  62.1% |  5.5%  |
|jetty   |-n 20000 -c 50 |  67.8% |  2.5%  |
|jetty   |-n 20000 -c 100|  64.2% |  3.2%  |
|jetty   |-n 20000 -c 200|  65.9% |  3.6%  |
|undertow|-n 20000 -c 50 |  67.5% |  2.4%  |
|undertow|-n 20000 -c 100|  63.3% |  2.7%  |
|undertow|-n 20000 -c 200|  55.2% |  3.1%  |

### 3.3 3.2 第三轮简单对比

|对比条件\服务器|tomcat  |jetty   |undertow             |
|---          |---     |---     |---                  |
|打包时间      |2.848 s |4.218 s |2.969 s              |
|编译时间      |0.027 s |0.628 s |0.061 s              |
|项目jar包大小 |16.8  MB|16.7 MB  |16.9 MB              |
|starter大小  |404    B|404    B |406    B             |   
|内存最大占用   |340   MB|120   MB|41    MB             |
|启动时间      |1.524 s |1.395 s |1.429 s              |
|响应头        | -      |     -  |Connection:keep-alive|

四、配置升级
------

|服务器   |参数数优化   |
|---     |---        |
|tomcat  |最大线程数400|
|jetty   |最大线程数400|
|undertow|最大线程数400|
|webAsync|最大线程数400|

配置如下

- tomcat配置

``` yml
server:
  port: 8081
  name: tomcat
  tomcat:
    max-threads: 400
springmvc:
  thread:
    core: 10
    max: 400
    queue: 200
```
- jetty配置

``` yml
server:
  port: 8082
  name: jetty
springmvc:
    thread:
      core: 10
      max: 400
      queue: 200
```

``` java
package net.csdn.container.jetty.config;

import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.util.thread.QueuedThreadPool;
import org.springframework.boot.web.embedded.jetty.JettyServerCustomizer;
import org.springframework.boot.web.embedded.jetty.JettyServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @Author： yaxuSong
 * @Description：
 * @Date： 19-8-9 上午10:51
 * @MOdified by:
 **/
@Configuration
public class JettyConfig {

    @Bean
    public JettyServletWebServerFactory jettyServletWebServerFactory(
            JettyServerCustomizer jettyServerCustomizer) {
        JettyServletWebServerFactory factory = new JettyServletWebServerFactory();
        factory.addServerCustomizers(jettyServerCustomizer);
        return factory;
    }

    @Bean
    public JettyServerCustomizer jettyServerCustomizer() {
        return this::threadPool;
    }

    private void threadPool(Server server) {
        // Tweak the connection config used by Jetty to handle incoming HTTP
        // connections
        final QueuedThreadPool threadPool = server.getBean(QueuedThreadPool.class);
        // 默认最大线程连接数200
        threadPool.setMaxThreads(100);
        // 默认最小线程连接数8
        threadPool.setMinThreads(20);
        // 默认线程最大空闲时间60000ms
        threadPool.setIdleTimeout(60000);
    }
}
```
- undertow配置

``` yml
server:
  port: 8083
  name: undertow
  undertow:
    io-threads: 16
    worker-threads: 400
springmvc:
  thread:
    core: 10
    max: 400
    queue: 200
```
- 公共配置

``` java
package net.csdn.container.tomcat.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.web.servlet.config.annotation.AsyncSupportConfigurer;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

/**
 * @Author： yaxuSong
 * @Description：
 * @Date： 19-8-9 上午11:21
 * @MOdified by:
 **/
@Configuration
public class SpringMvcConfig extends WebMvcConfigurerAdapter {
    @Value("${springmvc.thread.core}")
    private Integer core;
    @Value("${springmvc.thread.max}")
    private Integer max;
    @Value("${springmvc.thread.queue}")
    private Integer queue;

    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer asyncSupportConfigurer) {
        ThreadPoolTaskExecutor threadPool = new ThreadPoolTaskExecutor();
        threadPool.setCorePoolSize(this.core);
        threadPool.setMaxPoolSize(this.max);
        threadPool.setQueueCapacity(this.queue);
        threadPool.initialize();
        asyncSupportConfigurer.setTaskExecutor(threadPool);
    }
}
```
五、针对配置后环境测试
----------

###### 使用seige进行的http同步测试

访问情况统计

|服务器   |条件        |命中   |成功率|吞吐量            |平均耗时|
|---     |---        |---   |---  |---              |---   |
|tomcat  |-c 100 -t 1|88764 |100% |1489.83 trans/sec|0.07  |
|tomcat  |-c 200 -t 1|108533|100% |1811.60 trans/sec|0.11  |
|tomcat  |-c 400 -t 1|122054|100% |2059.98 trans/sec|0.19  |
|jetty   |-c 100 -t 1|106941|100% |1965.06 trans/sec|0.06  |
|jetty   |-c 200 -t 1|124423|100% |2079.26 trans/sec|0.10  |
|jetty   |-c 400 -t 1|150627|100% |2507.11 trans/sec|0.16  |
|undertow|-c 100 -t 1|154914|100% |2618.56 trans/sec|0.04  |
|undertow|-c 200 -t 1|124148|100% |2073.28 trans/sec|0.10  |
|undertow|-c 400 -t 1|141810|100% |2397.87 trans/sec|0.16  |

开启线程统计

|服务器   |条件         |存活线程数|守护线程数|累计开启线程数|
|---     |---        |---     |---     | ---         |
|tomcat  |初始        |   31   |    27  |41           |
|tomcat  |-c 100 -t 1|   147  |    143 |161          |
|tomcat  |-c 200 -t 1|   179  |    175 |312          |
|tomcat  |-c 400 -t 1|   204  |    200 |488          |
|jetty   |初始        |   36   |    14  |46           |
|jetty   |-c 100 -t 1|  115   |    12  |127          |
|jetty   |-c 200 -t 1|  115   |    13  |129          |
|jetty   |-c 400 -t 1|  112   |    11  |131          |
|undertow|初始        |  32    |    14  |42           |
|undertow|-c 100 -t 1|  430   |    12  |443          |
|undertow|-c 200 -t 1|  430   |    12  |444          |
|undertow|-c 400 -t 1|  430   |    12  |443          |

cpu和内存统计

|服务器   |条件        |CPU占用率|内存占用率|
|---     |---        |---     |---     |
|tomcat  |初始        |  0.7%  |  1.3%  |
|tomcat  |-c 100 -t 1|  48.2% |  3.2%  |
|tomcat  |-c 200 -t 1|  49.9% |  2.8%  |
|tomcat  |-c 400 -t 1|  50.5% |  7.4%  |
|jetty   |初始        |  1.3%  |  1.2%  |
|jetty   |-c 100 -t 1|  51.9% |  3.5%  |
|jetty   |-c 200 -t 1|  55.9% |  3.3%  |
|jetty   |-c 400 -t 1|  52.8% |  4.4%  |
|undertow|初始        |  0.3%  |  1.5%  |
|undertow|-c 100 -t 1|  15.6% |  4.2%  | 
|undertow|-c 200 -t 1|  52.5% |  3.9%  |
|undertow|-c 400 -t 1|  51.9% |  3.2%  | 

###### 使用seige进行的http异步测试

访问情况统计

|服务器   |条件        |命中   |成功率|吞吐量            |平均耗时|
|---     |---        |---   |---  |---              |---   |
|tomcat  |-c 100 -t 1|74207 |100% |1240.30 trans/sec|0.08  |
|tomcat  |-c 200 -t 1|80506 |100% |1351.45 trans/sec|0.15  |
|tomcat  |-c 400 -t 1|88399 |100% |1483.95 trans/sec|0.27  |
|jetty   |-c 100 -t 1|95687 |100% |1607.10 trans/sec|0.06  |
|jetty   |-c 200 -t 1|134072|100% |2268.18 trans/sec|0.09  |
|jetty   |-c 400 -t 1|117788|100% |1981.63 trans/sec|0.20  |
|undertow|-c 100 -t 1|92402 |100% |1561.11 trans/sec|0.06  |
|undertow|-c 200 -t 1|95865 |100% |1598.02 trans/sec|0.12  |
|undertow|-c 400 -t 1|102129|100% |1719.92 trans/sec|0.23  |

开启线程统计

|服务器   |条件         |存活线程数|守护线程数|累计开启线程数|
|---     |---        |---     |---     | ---         |
|tomcat  |-c 100 -t 1|   146  |    136 |161          |
|tomcat  |-c 200 -t 1|   225  |    211 |348          |
|tomcat  |-c 400 -t 1|   520  |    416 |1078         |
|jetty   |-c 100 -t 1|   128  |    15  |138          |
|jetty   |-c 200 -t 1|   124  |    11  |139          |
|jetty   |-c 400 -t 1|   125  |    12  |141          |
|undertow|-c 100 -t 1|   441  |    13  |454          |
|undertow|-c 200 -t 1|   440  |    12  |454          |
|undertow|-c 400 -t 1|   440  |    12  |455          |

cpu和内存统计

|服务器   |条件        |CPU占用率|内存占用率|
|---     |---        |---     |---     |
|tomcat  |-c 100 -t 1|  58.4% |  4.4%  |
|tomcat  |-c 200 -t 1|  56.5% |  6.4%  |
|tomcat  |-c 400 -t 1|  54.5% |  7.9%  |
|jetty   |-c 100 -t 1|  80.5% |  5.3%  |
|jetty   |-c 200 -t 1|  56.2% |  4.5%  |
|jetty   |-c 400 -t 1|  52.8% |  4.4%  |
|undertow|-c 100 -t 1|  67.2% |  3.1%  |
|undertow|-c 200 -t 1|  58.9% |  3.0%  |
|undertow|-c 400 -t 1|  55.2% |  2.8%  |

六、总结
------------

这些数字表明undertow在性能和内存使用方面表现确实相对其他两个要好。

七、参考文档
-------------

1. Tomcat vs. Jetty vs. Undertow: Comparison of Spring Boot Embedded Servlet Containers:[https://examples.javacodegeeks.com/enterprise-java/spring/tomcat-vs-jetty-vs-undertow-comparison-of-spring-boot-embedded-servlet-containers/](https://examples.javacodegeeks.com/enterprise-java/spring/tomcat-vs-jetty-vs-undertow-comparison-of-spring-boot-embedded-servlet-containers/)
2. Embedded Servlet Container Support:[https://docs.spring.io/spring-boot/docs/2.0.2.RELEASE/reference/htmlsingle/#boot-features-embedded-container](https://docs.spring.io/spring-boot/docs/2.0.2.RELEASE/reference/htmlsingle/#boot-features-embedded-container)
3. Embedded Web Servers:[https://docs.spring.io/spring-boot/docs/2.0.2.RELEASE/reference/htmlsingle/#howto-embedded-web-servers](https://docs.spring.io/spring-boot/docs/2.0.2.RELEASE/reference/htmlsingle/#howto-embedded-web-servers)
4. 后续之《SpringBoot服务器压测对比（jetty、tomcat、undertow）》:[https://my.oschina.net/shyloveliyi/blog/2980868?from=singlemessage](https://my.oschina.net/shyloveliyi/blog/2980868?from=singlemessage)
5. Spring Boot ：Undertow:[https://www.jianshu.com/p/e625b8aa0e80](https://www.jianshu.com/p/e625b8aa0e80)
6. Tomcat 、Jetty 和 Undertow 对比测试:[https://www.jianshu.com/p/f7cb40a8ce22](https://www.jianshu.com/p/f7cb40a8ce22)


