# RTCPeerConnection 构造

## 目录

<!-- vim-markdown-toc GFM -->

- [spec](#spec)
- [pion/webrtc@v1.2.0中RTCPeerConnection的实现](#pionwebrtcv120中rtcpeerconnection的实现)
- [最后](#最后)

<!-- vim-markdown-toc -->

## spec

[链接](https://www.w3.org/TR/webrtc/#constructor)

webrtc spec规定，构造时需要完成以下几个步骤，27个

- 下面步骤中，任务未提及的失败，都需要抛出一个UnknownError异常
  - 异常里的message要包含相应的描述
- 构造一个RTCPeerConnect实例
- 一个安全设置origin(html会用到，但一般webrtc api的实现并不需要)
  - 意思就是如果RTCPeerConnection有origin字段，初始化为当前对象的origin
- 如果配置里的证书列表不为空，则为每个证书执行以下流程：
  - 如果证书的过期时间(expires)小于当前时间，抛出一个OnvalidAccessError的异常
  - 如果证书的origin和连接的origin不一致，则说明不安全，抛出一个InvalidAccessError异常
  - 存储证书
- 如果配置的证书列表为空
  - 用RTCPeerConnection实例生成一个或多个RTCCertificate实例
  - 这个操作可能是异步的，所以在整数生成好之前，后续的步骤中可能显示未定义证书
  - 处于安全考虑，这里的证书不是使用pki证书，而是使用自签名证书
  - 这样保证了两点：不需要无限期使用密钥;不需要额外的证书检查
- 初始化ice agent连接
- 如果配置中的iceTransportPolicy未定义，初始化为all，表示所有类型的候选都收集
- 如果配置中的bundlePolicy未定义，初始化为balanced，平衡性的复用
- 如果配置中的rtcpMuxPolicy未定义，初始化为require,表示启用rtp收集，不为rtcp收集候选
- 如果有Configuration字段，指向配置对象
- 如果有IsClosed字段，初始化为false
- 如果有NegotiationNeeded字段，初始化为false
- 如果有SctpTransport字段，初始化为空
- 如果有Operations字段，初始化为空，这个表示操作链
- 如果有LastCreatedOffer字段，初始化为""
- 如果有LastCreatedAnswer字段，初始化为""
- 如果有EarlyCandidates字段，初始化为空列表
- 将信令状态置为stable，将连接状态/ice连接状态/ice收集状态置为new
- 如果有sdp字段，全部设置为空，因为是指针
  - PendingLocalDescription
  - CurrentLocalDescription
  - PendingRemoteDescription
  - CurrentRemoteDescription
- 如果有LocalIceCredentialsToReplace字段，初始化为空
- 返回连接实例(RTCPeerConnection)

## pion/webrtc@v1.2.0中RTCPeerConnection的实现

    func New(configuration RTCConfiguration) (*RTCPeerConnection, error) {
      pc := RTCPeerConnection{
        configuration: RTCConfiguration{
          IceServers:           []RTCIceServer{},
          IceTransportPolicy:   RTCIceTransportPolicyAll,
          BundlePolicy:         RTCBundlePolicyBalanced,
          RtcpMuxPolicy:        RTCRtcpMuxPolicyRequire,
          Certificates:         []RTCCertificate{},
          IceCandidatePoolSize: 0,
        },
        isClosed:          false,
        negotiationNeeded: false,
        lastOffer:         "",
        lastAnswer:        "",
        SignalingState:    RTCSignalingStateStable,
        // IceConnectionState: RTCIceConnectionStateNew, // FIXME SWAP-FOR-THIS
        IceConnectionState: ice.ConnectionStateNew, // FIXME REMOVE
        IceGatheringState:  RTCIceGatheringStateNew,
        ConnectionState:    RTCPeerConnectionStateNew,
        mediaEngine:        DefaultMediaEngine,
        sctpTransport:      newRTCSctpTransport(),
        dataChannels:       make(map[uint16]*RTCDataChannel),
      }

      var err error
      if err = pc.initConfiguration(configuration); err != nil {
        return nil, err
      }

      var urls []*ice.URL
      for _, server := range pc.configuration.IceServers {
        for _, rawURL := range server.URLs {
          var url *ice.URL
          url, err = ice.ParseURL(rawURL)
          if err != nil {
            return nil, err
          }

          urls = append(urls, url)
        }
      }

      pc.networkManager = network.NewManager(urls, pc.generateChannel, pc.iceStateChange)

      return &pc, nil
    }

这个实现基本符合spec中的规定，只是少了:

- origin字段
- EarlyCandidates字段
- LocalIceCredentialsToReplace字段

下面看看initConfiguration()，下面就不贴代码了，对着源码看

- 这个函数的描述是：对RTCConfiguration进行校验，并初始化一些值
- 这个函数和SetConfiguration有些不同，这个函数仅作为初始化检测，检测的项少一些

ps:RTCPeerConnection中的RTCConfiguration字段是值，不是引用。
在New()，RTCPeerConnection实例化时，只是简单初始化了一个RTCConfiguration对象，
并未从入参拷贝任何值。

流程如下：

- 拷贝peer的标识名
- 如果有509证书
  - 判断证书是否过期，过期就报InvalidAccessError异常(符合spec)
  - 没有过期的拷贝到RTCPeerConnection.RTCConfigurate
- 没有证书
  - 利用Go的标准库cryto，生成一个证书,因为重心不在加解密，所以暂时不深入生成证书的过程
- 拷贝bundle策略/rtcp复用策略/ice候选策略
- 拷贝ice候选池的大小
- 校验ice服务列表，并拷贝

校验ice服务列表很有意思，在rtciceserver.go，RTCIceServer.validate(),
遍历所有的服务url，按不同的证书类型(这里的证书是指ice服务访问整数)，
解析后，查看是否有解析失败的。之前的文章也提到了，ice服务证书有两种：
密码 + OAuth2.0。

从initConfiguration()回到New()构造函数，
构造函数的后面一节是从配置中提取出ice的url列表，然后构造出一个network.Manager对象。

## 最后

pion/webrtc@v1.2.0， 对于RTCPeerConnection的构造是符合webrtc spec的。
