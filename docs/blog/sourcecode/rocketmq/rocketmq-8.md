# Broker存储消息-1

## 发送消息

```
    private void sendMessageAsync(
        final String addr,
        final String brokerName,
        final Message msg,
        final long timeoutMillis,
        final RemotingCommand request,
        final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
        final MQClientInstance instance,
        final int retryTimesWhenSendFailed,
        final AtomicInteger times,
        final SendMessageContext context,
        final DefaultMQProducerImpl producer
    ) throws InterruptedException, RemotingException {
        final long beginStartTime = System.currentTimeMillis();
        this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
            // 发送成功回调
            @Override
            public void operationComplete(ResponseFuture responseFuture) {
                long cost = System.currentTimeMillis() - beginStartTime;
                RemotingCommand response = responseFuture.getResponseCommand();
                // 不设置回调函数的处理方式
                if (null == sendCallback && response != null) {

                    try {
                        // 发送结果
                        SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response, addr);
                        // 存在发送结果并且发送上下文不为空，将发送结果set到上下文中并执行发送消息后钩子
                        if (context != null && sendResult != null) {
                            context.setSendResult(sendResult);
                            context.getProducer().executeSendMessageHookAfter(context);
                        }
                    } catch (Throwable e) {
                    }
                    // 更新容错信息
                    producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                    return;
                }
                // 设置回调函数的处理方式
                if (response != null) {
                    try {
                        SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response, addr);
                        assert sendResult != null;
                        if (context != null) {
                            context.setSendResult(sendResult);
                            context.getProducer().executeSendMessageHookAfter(context);
                        }

                        try {
                            // 回调函数设置发送结果
                            sendCallback.onSuccess(sendResult);
                        } catch (Throwable e) {
                        }
                        // 更新容错信息
                        producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                    } catch (Exception e) {
                        producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                        // 异常处理
                        onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                            retryTimesWhenSendFailed, times, e, context, false, producer);
                    }
                } else {
                    // 更新容错信息
                    producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                    if (!responseFuture.isSendRequestOK()) {
                        // 发送失败
                        MQClientException ex = new MQClientException("send request failed", responseFuture.getCause());
                        onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                            retryTimesWhenSendFailed, times, ex, context, true, producer);
                    } else if (responseFuture.isTimeout()) {
                        // 超时
                        MQClientException ex = new MQClientException("wait response timeout " + responseFuture.getTimeoutMillis() + "ms",
                            responseFuture.getCause());
                        onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                            retryTimesWhenSendFailed, times, ex, context, true, producer);
                    } else {
                        // 不知道原因
                        MQClientException ex = new MQClientException("unknow reseaon", responseFuture.getCause());
                        onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                            retryTimesWhenSendFailed, times, ex, context, true, producer);
                    }
                }
            }
        });
    }
```

- 发送消息命令code

public static final int SEND_MESSAGE = 10;

## Broker接收生产者发送消息请求

&nbsp; &nbsp; Broker端启动后注册了请求处理器`SendMessageProcessor`，`SendMessageProcessor#asyncProcessRequest`处理请求，Broker消息存储有三个主要的类，分别是：CommitLog，MappedFileQueue，MappedFile；三个对象的比例是1:1:N，即一个日志文件对应一个文件对应多个消息存储文件。