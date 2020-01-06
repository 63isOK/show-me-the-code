# p2p连接状态的更新

## spec

[spec](https://www.w3.org/TR/webrtc/#update-the-connection-state)

RTCPeerConnection中有一个字段表示p2p连接状态，
每当RTCDtlsTransport状态变更或isClosed字段变为true(连接关闭了)，
这个p2p连接状态都会被更新，具体流程如下：

- 获取连接对象
- 将新状态映射到RTCPeerConnectionState枚举中的一个
- 如果新状态和连接对象的状态一致，退出(没有必要进行变更)
- 如果不一致，就将连接对象的状态进行更新
- 并触发一个事件，connectionstatechange的事件
