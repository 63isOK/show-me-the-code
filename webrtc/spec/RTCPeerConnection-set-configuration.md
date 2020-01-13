# 设置配置

配置configuration很重要，因为她是构造连接的参数

[webrtc spec](https://www.w3.org/TR/webrtc/#dom-rtcpeerconnection-setconfiguration)

## spec

这个函数主要是更新RTCPeerConnection的构造参数RTCConfiguration

按jsep的规定，这里的配置修改包括了ice agent的配置。

当ice配置变更时，那就需要重新进行ice协商，此时或许需要进行ice restart

流程如下：

- 连接断开，报一个InvalidStateError错误
- 执行具体的变更流程

具体的配置变更流程参考 4.4.1.6

## pion/webrtc@v1.2.0

SetConfiguration的流程：

- 如果连接关闭，退出
- 检查perr的标识，新加功能(后续可能会有更多实现)
- 检查证书是否一致
- 检查bundle/rtcpMux策略
- 检查ice候选池大小
- 检查ice候选策略
- 最后校验ice服务的有效性

这其中只是简单的替换了连接中的配置属性，并没有太多的业务支持(比如说ice restart)
