# RTCPeerConnection 构造

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
