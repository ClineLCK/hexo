---
title: ElasticSearch 安装注意事项
date: 2018-05-31 16:07:10
tags: ElasticSearch
categories: ElasticSearch

---

### 安装异常报错

> max file descriptors [65535] for elasticsearch process is too low, increase to at least [65536]

修改 /etc/security/limits.conf ，在文件最后添加或者修改  * hard nofile 65536

> max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

修改 /etc/sysctl.conf  ,在文件最后添加  vm.max_map_count=262144


> system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk

修改 /elasticsearch-6.2.4/config/elasticsearch.yml ，在文件最后添加 bootstrap.system_call_filter: false

### es运行内存修改

修改 /elasticsearch-6.2.4/config/jvm.options

```
################################################################
## IMPORTANT: JVM heap size
################################################################
##
## You should always set the min and max JVM heap
## size to the same value. For example, to set
## the heap to 4 GB, set:
##
## -Xms4g
## -Xmx4g
##
## See https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html
## for more information
##
################################################################

# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space

-Xms512m
-Xmx512m


```



