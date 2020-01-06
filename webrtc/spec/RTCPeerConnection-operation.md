# RTCPeerConnection相关操作

## 构造函数

要通过配置对象来生成一个RTCPeerConnection对象

    func New(configuration RTCConfiguration) (*RTCPeerConnection, error) {}

上面就是pion/webrtc@v1.2.0中的定义，符合标准

## 配置对象的servers信息

配置对象的servers要包含ice服务器的信息，
应用程序应该可以为每种ice类型配置多个服务器，
turn服务器也可以充当stun服务器来获取公网地址。

因为现在大多数turn服务都实现了stun协议，例如coturn。

    type RTCConfiguration struct {
      IceServers []RTCIceServer
      IceTransportPolicy RTCIceTransportPolicy
      BundlePolicy RTCBundlePolicy
      RtcpMuxPolicy RTCRtcpMuxPolicy
      PeerIdentity string
      Certificates []RTCCertificate
      IceCandidatePoolSize uint8
    }

spec指定的servers就是上面结构中的IceServers

## RTCPeerConnection的4个状态信息

RTCPeerConnection包含4个状态信息：

- 信令状态，表示offer/answer协商的状态
- 连接状态，表示p2p连接状态
- ice候选收集状态
- ice连接状态

在webrtc spec中，这4个状态分别都有单独的篇章来讲述。
这4个状态，都是在对象被构造时进行初始化的。

    type RTCPeerConnection struct {
      sync.RWMutex
      configuration RTCConfiguration
      CurrentLocalDescription *RTCSessionDescription
      PendingLocalDescription *RTCSessionDescription
      CurrentRemoteDescription *RTCSessionDescription
      PendingRemoteDescription *RTCSessionDescription
      SignalingState RTCSignalingState
      IceGatheringState RTCIceGatheringState // FIXME NOT-USED
      IceConnectionState ice.ConnectionState // FIXME REMOVE
      ConnectionState RTCPeerConnectionState
      idpLoginURL *string

      isClosed          bool
      negotiationNeeded bool

      lastOffer  string
      lastAnswer string

      // Media
      mediaEngine     *MediaEngine
      rtpTransceivers []*RTCRtpTransceiver

      // sctpTransport
      sctpTransport *RTCSctpTransport

      // DataChannels
      dataChannels map[uint16]*RTCDataChannel

      onSignalingStateChangeHandler     func(RTCSignalingState)
      onICEConnectionStateChangeHandler func(ice.ConnectionState)
      onTrackHandler                    func(*RTCTrack)
      onDataChannelHandler              func(*RTCDataChannel)

      // Deprecated: Internal mechanism which will be removed.
      networkManager *network.Manager
    }

可以看到，pion/webrtc@v1.2.0里，这个对象确实包含了4个状态。
在New()构造函数中，初始化实例时，这4个字段都被初始化为对应的初始状态了。

## ice agent

两端之间有一个叫法，本端叫agent，远端叫peer，不管是p2p，还是ice都是这么叫的。

RTCPeerConnectin中如果实现了ice协议，那么RTCPeerConnection也叫ice agent。
RTCPeerConnection中有些方法就是用来处理ice的：

- addIceCandidate
- setConfiguration
- setLocalDescription
- setRemoteDescription
- close

这几个交互方法，是在jsep中定义的。除此之外，当RTCIceTransport内部状态变更时，
ice agent也会和用户进行交互，具体在讨论RTCIceTransport时再深入讨论。

从上面的RTCPeerConnection结构体中知道，有个字段是network.Magager,
之前也分析过了，这个表示了ice连接，并且还复用了dtls/srtp，
RTCPeerConnection就是利用这个字段实现的ice，下面列出jsep定义的几个方法：

- AddIceCandidate
- SetConfiguration
- SetLocalDescription
- SetRomoteDescription
- Close

其实jsep在RTCPeerConnection的扩展方法还有很多，不过webrtc spec中只吸纳了这些，
后续估计会进一步完善，兼容jsep。

## note, sdp协商状态

      CurrentLocalDescription *RTCSessionDescription
      PendingLocalDescription *RTCSessionDescription
      CurrentRemoteDescription *RTCSessionDescription
      PendingRemoteDescription *RTCSessionDescription

这是RTCPeerConnection的4个sdp，sdp协商状态用这4个sdp外加一个状态表示，
webrtc spec中用p2p连接状态来表示，我现在的理解是用信令状态，
已经提交了[issuse](https://github.com/w3c/webrtc-pc/issues/2430)，
看pion/webrtc源码，都是信令状态，所以有点疑惑。

至于是哪个状态，可以看issues的最终结果，反正现在sdp的协商状态由这5个变量表示。
4个sdp只能由setLocalDescription/setRemoteDescription函数来修改，
在addIceCandidate中和ice处理过程中也能进行修改。有很多事件也在监听这5个变量，
webrtc spec规定，这5个变量是先变更，再触发相应的事件，所以不会出现并发问题。
