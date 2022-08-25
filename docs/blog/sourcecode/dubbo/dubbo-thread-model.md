# Dubbo线程模型 <!-- {docsify-ignore-all} -->

## Dubbo协议下的服务端线程模型

&nbsp; &nbsp; Dubbo协议的网络框架使用的是Netty，Dubbo对Channel上的操作抽象成了5中行为，分别是：建立连接、断开连接、发送消息、接收消息、异常捕获。

&nbsp; &nbsp; Dubbo框架的线程模型与以上这五种行为息息相关，Dubbo协议Provider线程模型可以分为五类，也就是AllDispatcher、DirectDispatcher、MessageOnlyDispatcher、ExecutionDispatcher、ConnectionOrderedDispatcher。

