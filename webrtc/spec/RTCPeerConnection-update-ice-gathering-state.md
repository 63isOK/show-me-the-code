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
