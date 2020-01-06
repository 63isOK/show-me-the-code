# p2p连接的介绍

此处p2p连接指RTCPeerConnection，以及周边相关的数据结构，
但核心还是RTCPeerConnection，对外，p2p连接指agent和peer之间的连接。

这里的agnet/peer可以是浏览器(目前大部分流程器都实现了webrtc标准)，
也可以是其他实现了webrtc标准的其他程序。

这里面的通信，是指在信令通道，进行控制信令的协作交换的过程，
spec里并未对信令通道的实现做具体指定，不过一般由server提供，
一般走xhr(XMLHttpRequest)或websocket。
