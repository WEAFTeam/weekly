---
title: Elasticsearch在Linux的安装与使用
description: Elasticsearch在Linux的安装与使用
tags:
  - Linux
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/elk.png'
category: Linux
date: '2019-08-07 21:49:23'
---
一、简介
=========
ElasticSearch 是一个分布式、高扩展、高实时的搜索与数据分析引擎。ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java语言开发的，并作为Apache许可条款下的开放源码发布，是一种流行的企业级搜索引擎。ElasticSearch用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。官方客户端在Java、.NET（C#）、PHP、Python、Apache Groovy、Ruby和许多其他语言中都是可用的。根据DB-Engines的排名显示，Elasticsearch是最受欢迎的企业搜索引擎，其次是Apache Solr，也是基于Lucene。

二、下载安装
===========
首先我们去官网下载我们所需的文件

- [Download Elasticsearch-https://www.elastic.co/cn/downloads/elasticsearch](https://www.elastic.co/cn/downloads/elasticsearch)
- [Download Kibana:https://www.elastic.co/cn/downloads/kibana](https://www.elastic.co/cn/downloads/kibana)

在现在页面我们可以选择Not the version you're looking for? View **past releases**中的黑色字体来下载历史版本.

我这边项目中使用的是6.3.2版本，所以这里我也选择下载6.3.2

下载完两个文件分别是
- kibana-6.3.2-linux-x86_64.tar.gz
- elasticsearch-6.3.2.zip
使用以下命令分别解压

``` shell
tar -zxvf kibana-6.3.2-linux-x86_64.tar.gz
unzip elasticsearch-6.3.2.zip
```
一般 kibana是无需设置的，但是启动的时候，我们需要先启动elasticsearch那样，他会自动链接和配置
这里我们打开elasticsearch的下的config文件夹下的elasticsearch.yml
将以下注解打开并配置

``` yml
cluster.name:songyaxu-ubuntu
node.name:songyaxu-node-1
# 其他可选参数
network.host: 0.0.0.0
http.port: 9200
transport.tcp.port: 9300
```
如果是一些集群配置这里还需配置一些其他参数，具体参考
- [Elasticsearch: 权威指南:https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)
- [Elasticsearch: 权威指南:https://es.xiaoleilu.com/](https://es.xiaoleilu.com/)

先后启动kibana和elasticsearch
``` shell
/app/elasticsearch-6.3.2/bin/elasticsearch

/app/kibana-6.3.2-linux-x86_64/bin/kibana
```

三、使用java操作elasticsearch
========

### 3.1 实现方式一，自己通过原始api进行操作
``` java
package com.songyaxu.community.config;

import com.songyaxu.common.Converter;
import com.songyaxu.common.json.JsonConverterHolder;
import org.elasticsearch.client.Client;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.transport.TransportAddress;
import org.elasticsearch.transport.client.PreBuiltTransportClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.net.InetAddress;
import java.net.UnknownHostException;

/**
 * @author yaxuSong
 * @since 19/6/4  am10:55
 */
@Configuration
public class ElasticSearchConfig {
    private static final Logger logger = LoggerFactory.getLogger(FullTextSearchConfig.class);

    @Value("${elastic.host}")
    private String elasticHost;
    
    @Value("${elastic.port}")
    private int elasticPort;

    @Value("${elastic.cluster.name}")
    private String elasticClusterName;
    
    @Bean
    public Client elasticsearchClient() {
        if (elasticHost != null) {
            try {
                Settings settings = Settings.builder().put("cluster.name", elasticClusterName).build();
                return new PreBuiltTransportClient(settings).addTransportAddress(new TransportAddress(InetAddress.getByName(elasticHost), elasticPort));

            } catch (UnknownHostException e) {
                logger.error("failed to connect ES server!!!", e);
            }
        }
        return null;
    }

    @Bean
    public Converter jsonConverter() {
        return JsonConverterHolder.getInstance().getConverter();
    }
    
}

```
具体实现我们这边可以通过使用org.elasticsearch.client提供的方法进行操作。为了方便这边也可以自己在封装一层。

### 3.2 实现方式二，通过springdata提供方法进行

这里操作就非常简单了，我们只需使用springboot的自动配置方式，填写一下配置参数
``` yml
spring:
  data:
    elasticsearch:
      cluster-name: songyaxu-ubuntu
      cluster-nodes: 127.0.0.1:9300
      repositories:
        enabled: true
```

操作时候我们可以通过注入以下模板进行操作

``` java
@Autowired
private ElasticsearchTemplate elasticsearchTemplate;
```

四、Kibana的使用
========

Kibana 是以web的形式展现给用户的，我们可以通过访问**http://localhost:5601**来进行查看

一些具体的使用请查看官方文档[官方指南：https://www.elastic.co/guide/en/kibana/current/getting-started.html](https://www.elastic.co/guide/en/kibana/current/getting-started.html)


五、参考文章
=========

1. Elasticsearch: 权威指南:[https://es.xiaoleilu.com/](https://es.xiaoleilu.com/)
2. Connecting to Elasticsearch by Using Spring Data[https://docs.spring.io/spring-boot/docs/2.0.2.RELEASE/reference/htmlsingle/#boot-features-elasticsearch](https://docs.spring.io/spring-boot/docs/2.0.2.RELEASE/reference/htmlsingle/#boot-features-elasticsearch)
3. ikana官方指南：[https://www.elastic.co/guide/en/kibana/current/getting-started.html](https://www.elastic.co/guide/en/kibana/current/getting-started.html)
