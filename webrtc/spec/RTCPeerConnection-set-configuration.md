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
