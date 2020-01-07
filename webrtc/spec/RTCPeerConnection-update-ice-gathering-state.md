# ice候选收集状态的变更

[spec](https://www.w3.org/TR/webrtc/#update-the-ice-gathering-state)

## spec

ice候选收集的状态变更要走以下几步：

- 如果RTCPeerConnection.IsClosed是true，退出
- 将新状态用RTCIceGatheringState枚举值表示
- 如果新状态和老状态一致，退出
- 如果不一致，更新新状态
- 触发一个icegatheringstatechange事件
- 如果新状态是completed，还将触发一个icecandidate事件

## pion/webrtc@v1.2.0

源码中为候选的收集定义了3种状态：new/gathering/complete

对象构造时，收集候选的状态初始化为new

这个1.2.0版本中，并未对这个状态有其他修改，也并未对外暴露出收集状态变更的方式

在后续的版本中，应该会补全这块的，毕竟这个是非常重要的信息
