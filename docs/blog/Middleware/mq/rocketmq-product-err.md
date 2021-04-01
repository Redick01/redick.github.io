# 阿里云RocketMQ订阅关系一致性

https://help.aliyun.com/document_detail/43523.html?spm=a2c4g.11186623.6.732.52706142DyFCNR
订阅关系一致指的是同一个消费者Group ID下所有Consumer实例所订阅的Topic、Group ID、Tag必须完全一致。一旦订阅关系不一致，消息消费的逻辑就会混乱，甚至导致消息丢失。本文提供订阅关系一致的正确示例代码以及订阅关系不一致的错误示例代码，帮助您顺畅地订阅消息。