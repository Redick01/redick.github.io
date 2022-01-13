# Filebeat+Logstash收集多个日志发送到ES的不同索引 <!-- {docsify-ignore-all} -->

&nbsp; &nbsp; 在[Docker搭建ELK](/docs/blog/Middleware/elk/elk-1.md)章节中简单的介绍了机遇Docker搭建ELK系统，并提供一个例子，将nginx日志收集，存储到ES，然后通过kibana进行查询日志；本篇来介绍下，如何收集多个日志并发送到ES的不同索引。


## 方案一

&nbsp; &nbsp; 方案一，在filebeat配置input时添加索引字段，然后传动logstash，logstash直接用这个字段当作es的索引字段发送到es。